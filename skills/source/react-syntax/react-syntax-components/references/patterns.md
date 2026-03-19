# Component Composition Patterns

> Reference file for `react-syntax-components`. Compound components, render props, and HOC patterns.

---

## Compound Components (Context-Based)

The compound component pattern creates a flexible API where related components share implicit state through Context. The parent owns the state; children consume it.

### Full Pattern

```tsx
import React, { createContext, useContext, useState, useCallback, type PropsWithChildren } from 'react';

// 1. Define shared state shape
interface AccordionContextValue {
  readonly openItems: ReadonlySet<string>;
  readonly toggle: (id: string) => void;
}

// 2. Create context with NO default -- forces provider usage
const AccordionContext = createContext<AccordionContextValue | null>(null);

// 3. Custom hook for safe consumption
function useAccordion(): AccordionContextValue {
  const context = useContext(AccordionContext);
  if (context === null) {
    throw new Error('Accordion compound components must be used within <Accordion>');
  }
  return context;
}

// 4. Root component -- owns state, provides context
interface AccordionProps {
  readonly allowMultiple?: boolean;
  readonly children: React.ReactNode;
}

function Accordion({ allowMultiple = false, children }: AccordionProps): React.ReactElement {
  const [openItems, setOpenItems] = useState<ReadonlySet<string>>(new Set());

  const toggle = useCallback((id: string) => {
    setOpenItems((prev) => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear();
        next.add(id);
      }
      return next;
    });
  }, [allowMultiple]);

  const value = React.useMemo(() => ({ openItems, toggle }), [openItems, toggle]);

  return (
    <AccordionContext value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext>
  );
}

// 5. Child components -- consume context
interface AccordionItemProps {
  readonly id: string;
  readonly children: React.ReactNode;
}

function AccordionItem({ id, children }: AccordionItemProps): React.ReactElement {
  const { openItems } = useAccordion();
  const isOpen = openItems.has(id);

  return (
    <div className={`accordion-item ${isOpen ? 'open' : ''}`} data-item-id={id}>
      {children}
    </div>
  );
}

interface AccordionTriggerProps {
  readonly itemId: string;
  readonly children: React.ReactNode;
}

function AccordionTrigger({ itemId, children }: AccordionTriggerProps): React.ReactElement {
  const { toggle } = useAccordion();

  return (
    <button className="accordion-trigger" onClick={() => toggle(itemId)}>
      {children}
    </button>
  );
}

interface AccordionContentProps {
  readonly itemId: string;
  readonly children: React.ReactNode;
}

function AccordionContent({ itemId, children }: AccordionContentProps): React.ReactElement | null {
  const { openItems } = useAccordion();
  if (!openItems.has(itemId)) return null;

  return <div className="accordion-content">{children}</div>;
}

// 6. Attach sub-components to root for clean API
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Usage
<Accordion allowMultiple>
  <Accordion.Item id="faq-1">
    <Accordion.Trigger itemId="faq-1">What is React?</Accordion.Trigger>
    <Accordion.Content itemId="faq-1">
      <p>React is a JavaScript library for building user interfaces.</p>
    </Accordion.Content>
  </Accordion.Item>
  <Accordion.Item id="faq-2">
    <Accordion.Trigger itemId="faq-2">What are hooks?</Accordion.Trigger>
    <Accordion.Content itemId="faq-2">
      <p>Hooks let you use state and other React features in function components.</p>
    </Accordion.Content>
  </Accordion.Item>
</Accordion>
```

### When to Use Compound Components

- Building reusable UI primitives (tabs, accordions, dropdown menus, dialogs)
- Consumer needs flexible control over layout and ordering of sub-components
- Internal state must be shared without prop drilling

### Rules

- **ALWAYS** throw an error in the context hook when used outside the provider
- **ALWAYS** memoize the context value to prevent unnecessary re-renders
- **ALWAYS** attach sub-components to the root component for a clean dot-notation API
- **NEVER** expose internal context outside the compound component module

---

## Render Props Pattern

The render props pattern delegates rendering control to the consumer by accepting a function that returns React elements.

### Function as Children

```tsx
interface MouseTrackerProps {
  readonly children: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps): React.ReactElement {
  const [position, setPosition] = React.useState({ x: 0, y: 0 });

  return (
    <div
      style={{ height: '100%' }}
      onMouseMove={(e) => setPosition({ x: e.clientX, y: e.clientY })}
    >
      {children(position)}
    </div>
  );
}

// Usage
<MouseTracker>
  {({ x, y }) => (
    <div>
      Mouse is at ({x}, {y})
    </div>
  )}
</MouseTracker>
```

### Named Render Prop

```tsx
interface DataFetcherProps<T> {
  readonly url: string;
  readonly render: (data: T | null, loading: boolean, error: string | null) => React.ReactNode;
}

function DataFetcher<T>({ url, render }: DataFetcherProps<T>): React.ReactElement {
  const [data, setData] = React.useState<T | null>(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState<string | null>(null);

  React.useEffect(() => {
    let ignore = false;
    setLoading(true);
    setError(null);

    fetch(url)
      .then((res) => res.json() as Promise<T>)
      .then((result) => {
        if (!ignore) {
          setData(result);
          setLoading(false);
        }
      })
      .catch((err: Error) => {
        if (!ignore) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => { ignore = true; };
  }, [url]);

  return <>{render(data, loading, error)}</>;
}

// Usage
<DataFetcher<User[]>
  url="/api/users"
  render={(data, loading, error) => {
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage message={error} />;
    return <UserList users={data!} />;
  }}
/>
```

### When to Use Render Props

- Consumer needs full control over what gets rendered with shared logic
- Building headless components (logic without UI)
- **ALWAYS** prefer custom hooks for pure logic sharing -- render props are for when you also need to wrap rendering

---

## Higher-Order Components (HOC)

HOCs wrap a component to inject additional props or behavior. **ALWAYS** prefer custom hooks for new code -- HOCs are harder to type and debug.

### Typed HOC Pattern

```tsx
interface WithAuthProps {
  readonly currentUser: User;
}

function withAuth<P extends WithAuthProps>(
  WrappedComponent: React.ComponentType<P>
): React.ComponentType<Omit<P, keyof WithAuthProps>> {
  function WithAuthComponent(props: Omit<P, keyof WithAuthProps>): React.ReactElement {
    const currentUser = useAuth(); // custom hook

    if (!currentUser) {
      return <Navigate to="/login" />;
    }

    // Type assertion required due to TypeScript limitation with HOCs
    return <WrappedComponent {...(props as P)} currentUser={currentUser} />;
  }

  // ALWAYS set displayName for DevTools debugging
  WithAuthComponent.displayName = `withAuth(${WrappedComponent.displayName ?? WrappedComponent.name ?? 'Component'})`;

  return WithAuthComponent;
}

// Usage
interface DashboardProps extends WithAuthProps {
  readonly title: string;
}

function Dashboard({ currentUser, title }: DashboardProps): React.ReactElement {
  return <h1>{title} - Welcome, {currentUser.name}</h1>;
}

const ProtectedDashboard = withAuth(Dashboard);
// <ProtectedDashboard title="Home" />  -- currentUser is injected
```

### HOC Rules

- **ALWAYS** set `displayName` on the wrapper component
- **NEVER** mutate the original component -- wrap it in a new component
- **NEVER** apply HOCs inside render -- ALWAYS apply at module scope
- **ALWAYS** forward all unknown props to the wrapped component
- **ALWAYS** prefer custom hooks over HOCs for new code

### When HOCs Are Still Useful

- Wrapping third-party components you cannot modify
- Adding cross-cutting behavior to many route-level components (e.g., authentication guards)
- Legacy codebases migrating incrementally to hooks

---

## Pattern Comparison

| Pattern | State Sharing | Render Control | TypeScript | Complexity |
|---------|--------------|----------------|------------|------------|
| Custom Hook | Logic only | None | Excellent | Low |
| Compound Component | Implicit via Context | Consumer arranges children | Good | Medium |
| Render Props | Via function args | Consumer controls output | Good | Medium |
| HOC | Via injected props | Wrapper controls | Difficult | High |

**Decision order** (ALWAYS follow this):
1. Custom hook -- if you only need shared logic
2. Compound component -- if you need a flexible multi-part UI
3. Render props -- if consumer needs full rendering control
4. HOC -- only as a last resort or for legacy integration
