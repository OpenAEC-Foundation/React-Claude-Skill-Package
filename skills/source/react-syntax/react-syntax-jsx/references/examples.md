# JSX Patterns and Code Examples

## Expressions in JSX

### Embedding Variables and Computations

```tsx
interface UserBadgeProps {
  user: { name: string; level: number };
}

function UserBadge({ user }: UserBadgeProps): JSX.Element {
  const greeting = `Welcome, ${user.name}`;
  const isVIP = user.level >= 10;

  return (
    <div className="badge">
      <h2>{greeting}</h2>
      <span>Level: {user.level}</span>
      <span>Status: {isVIP ? 'VIP' : 'Standard'}</span>
      <span>Points needed: {(10 - user.level) * 100}</span>
    </div>
  );
}
```

### Inline Styles (Object Expression)

```tsx
function ProgressBar({ percent }: { percent: number }): JSX.Element {
  return (
    <div
      style={{
        width: `${percent}%`,
        height: 8,
        backgroundColor: percent === 100 ? 'green' : 'blue',
        borderRadius: 4,
        transition: 'width 0.3s ease',
      }}
    />
  );
}
```

### Dynamic Class Names

```tsx
interface TabProps {
  label: string;
  isActive: boolean;
  onClick: () => void;
}

function Tab({ label, isActive, onClick }: TabProps): JSX.Element {
  return (
    <button
      className={`tab ${isActive ? 'tab--active' : ''}`}
      onClick={onClick}
      aria-selected={isActive}
    >
      {label}
    </button>
  );
}
```

---

## Conditional Rendering Patterns

### Early Return for Authorization

```tsx
interface ProtectedRouteProps {
  isAuthenticated: boolean;
  children: React.ReactNode;
}

function ProtectedRoute({ isAuthenticated, children }: ProtectedRouteProps): JSX.Element {
  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }
  return <>{children}</>;
}
```

### Ternary for Toggle States

```tsx
function ExpandableSection({ title, content }: { title: string; content: string }): JSX.Element {
  const [isExpanded, setIsExpanded] = useState<boolean>(false);

  return (
    <section>
      <button onClick={() => setIsExpanded(!isExpanded)}>
        {title} {isExpanded ? '(collapse)' : '(expand)'}
      </button>
      {isExpanded ? <p>{content}</p> : null}
    </section>
  );
}
```

### Safe && Pattern (Boolean Left Side)

```tsx
interface NotificationBadgeProps {
  count: number;
}

function NotificationBadge({ count }: NotificationBadgeProps): JSX.Element {
  return (
    <div className="icon">
      <BellIcon />
      {count > 0 && <span className="badge">{count}</span>}
    </div>
  );
}
```

### Variable Assignment for Multi-Branch Logic

```tsx
type Status = 'loading' | 'error' | 'empty' | 'success';

interface DataViewProps {
  status: Status;
  data: string[];
  error: Error | null;
}

function DataView({ status, data, error }: DataViewProps): JSX.Element {
  let content: JSX.Element;

  switch (status) {
    case 'loading':
      content = <Spinner />;
      break;
    case 'error':
      content = <ErrorMessage message={error?.message ?? 'Unknown error'} />;
      break;
    case 'empty':
      content = <EmptyState message="No results found" />;
      break;
    case 'success':
      content = (
        <ul>
          {data.map((item) => (
            <li key={item}>{item}</li>
          ))}
        </ul>
      );
      break;
  }

  return <div className="data-view">{content}</div>;
}
```

### IIFE Pattern for Inline Complex Logic

```tsx
function StatusMessage({ code }: { code: number }): JSX.Element {
  return (
    <div>
      {(() => {
        if (code >= 500) return <span className="error">Server Error</span>;
        if (code >= 400) return <span className="warning">Client Error</span>;
        if (code >= 200) return <span className="success">OK</span>;
        return <span>Unknown</span>;
      })()}
    </div>
  );
}
```

---

## Lists and Keys

### Basic List Rendering

```tsx
interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

function ProductList({ products }: { products: Product[] }): JSX.Element {
  return (
    <div className="product-grid">
      {products.map((product) => (
        <div key={product.id} className="product-card">
          <h3>{product.name}</h3>
          <p>${product.price.toFixed(2)}</p>
          {product.inStock ? (
            <button>Add to Cart</button>
          ) : (
            <span className="out-of-stock">Out of Stock</span>
          )}
        </div>
      ))}
    </div>
  );
}
```

### Filtered and Transformed Lists

```tsx
interface Person {
  id: number;
  name: string;
  profession: string;
}

function ScientistList({ people }: { people: Person[] }): JSX.Element {
  const scientists = people.filter((p) => p.profession === 'scientist');

  return (
    <ul>
      {scientists.map((person) => (
        <li key={person.id}>
          {person.name}
        </li>
      ))}
    </ul>
  );
}
```

### Nested Lists with Stable Keys

```tsx
interface Category {
  id: string;
  name: string;
  items: { id: string; label: string }[];
}

function CategoryList({ categories }: { categories: Category[] }): JSX.Element {
  return (
    <div>
      {categories.map((category) => (
        <section key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.items.map((item) => (
              <li key={item.id}>{item.label}</li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

### Fragment with Key in Lists

```tsx
import { Fragment } from 'react';

interface GlossaryEntry {
  id: string;
  term: string;
  definition: string;
}

function Glossary({ entries }: { entries: GlossaryEntry[] }): JSX.Element {
  return (
    <dl>
      {entries.map((entry) => (
        <Fragment key={entry.id}>
          <dt>{entry.term}</dt>
          <dd>{entry.definition}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

---

## TypeScript Component Patterns

### Props with Discriminated Unions

```tsx
type AlertProps =
  | { variant: 'success'; message: string }
  | { variant: 'error'; message: string; retryAction: () => void }
  | { variant: 'loading' };

function Alert(props: AlertProps): JSX.Element {
  switch (props.variant) {
    case 'success':
      return <div className="alert-success">{props.message}</div>;
    case 'error':
      return (
        <div className="alert-error">
          {props.message}
          <button onClick={props.retryAction}>Retry</button>
        </div>
      );
    case 'loading':
      return <div className="alert-loading"><Spinner /></div>;
  }
}
```

### Generic List Component

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => JSX.Element;
  keyExtractor: (item: T) => string;
  emptyMessage?: string;
}

function List<T>({ items, renderItem, keyExtractor, emptyMessage = 'No items' }: ListProps<T>): JSX.Element {
  if (items.length === 0) {
    return <p>{emptyMessage}</p>;
  }

  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  items={users}
  keyExtractor={(u) => u.id}
  renderItem={(u) => <span>{u.name} ({u.email})</span>}
  emptyMessage="No users found"
/>
```

### Polymorphic Component with `as` Prop

```tsx
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
  children: React.ReactNode;
} & Omit<React.ComponentPropsWithoutRef<E>, 'as' | 'children'>;

function Box<E extends React.ElementType = 'div'>({
  as,
  children,
  ...props
}: PolymorphicProps<E>): JSX.Element {
  const Component = as || 'div';
  return <Component {...props}>{children}</Component>;
}

// Usage
<Box>Default div</Box>
<Box as="section" id="main">Section element</Box>
<Box as="a" href="/about">Link element</Box>
```

---

## JSX Spread Attributes

### Rest/Spread for Wrapper Components

```tsx
interface TextFieldProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

function TextField({ label, error, ...inputProps }: TextFieldProps): JSX.Element {
  return (
    <div className="field">
      <label>
        {label}
        <input {...inputProps} aria-invalid={!!error} />
      </label>
      {error && <span className="field-error" role="alert">{error}</span>}
    </div>
  );
}

// All native input attributes pass through
<TextField
  label="Username"
  type="text"
  required
  minLength={3}
  maxLength={20}
  error={errors.username}
/>
```

### Composing Props from Multiple Sources

```tsx
interface BaseButtonProps {
  variant: 'primary' | 'secondary';
  size: 'sm' | 'md' | 'lg';
}

const sizeClasses: Record<BaseButtonProps['size'], string> = {
  sm: 'btn-sm',
  md: 'btn-md',
  lg: 'btn-lg',
};

function Button({
  variant,
  size,
  className,
  ...rest
}: BaseButtonProps & React.ButtonHTMLAttributes<HTMLButtonElement>): JSX.Element {
  return (
    <button
      className={`btn btn-${variant} ${sizeClasses[size]} ${className ?? ''}`}
      {...rest}
    />
  );
}
```

---

## Boolean Attributes and Self-Closing Tags

```tsx
function FormExample(): JSX.Element {
  const [isSubmitting, setIsSubmitting] = useState<boolean>(false);

  return (
    <form>
      {/* Boolean attribute: presence = true, omission = false */}
      <input type="text" required readOnly={false} />

      {/* Self-closing tags (no children) */}
      <input type="email" />
      <br />
      <hr />
      <img src="/logo.png" alt="Logo" />

      {/* Dynamic boolean */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

---

## PascalCase vs Lowercase

```tsx
// PascalCase: React treats as a custom component
function MyButton(): JSX.Element {
  return <button className="custom">Click</button>;
}

// Usage: PascalCase invokes the component
<MyButton />

// Lowercase: React treats as an HTML element
// <myButton /> would render an unknown HTML element, NOT the component above

// Dynamic component selection ALWAYS requires PascalCase variable
const components: Record<string, React.ComponentType> = {
  photo: PhotoStory,
  video: VideoStory,
};

function Story({ type }: { type: string }): JSX.Element {
  const SpecificStory = components[type]; // PascalCase variable
  return <SpecificStory />;
}
```
