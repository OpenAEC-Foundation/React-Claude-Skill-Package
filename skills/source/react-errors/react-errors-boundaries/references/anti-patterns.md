# Error Boundary Anti-Patterns

## Anti-Pattern 1: Using a Function Component as an Error Boundary

```typescript
// WRONG: Function components cannot be error boundaries
function ErrorBoundary({ children }: { children: React.ReactNode }) {
  const [hasError, setHasError] = useState(false);

  // This does NOT work — there is no hook equivalent for
  // getDerivedStateFromError or componentDidCatch
  useEffect(() => {
    // Cannot catch render errors here
  }, []);

  if (hasError) return <p>Error</p>;
  return <>{children}</>;
}
```

**WHY this is wrong**: React has NO hook equivalent for catching render-phase errors. The `getDerivedStateFromError` and `componentDidCatch` lifecycle methods are exclusive to class components. ALWAYS use a class component or the `react-error-boundary` library.

---

## Anti-Pattern 2: Catching Event Handler Errors with Boundaries

```typescript
// WRONG: Error boundaries do NOT catch event handler errors
<ErrorBoundary fallback={<p>Error</p>}>
  <button onClick={() => {
    throw new Error('Click failed');  // NOT caught by boundary
  }}>
    Click Me
  </button>
</ErrorBoundary>
```

**WHY this is wrong**: Event handlers execute outside the React render cycle. Error boundaries only intercept errors during rendering, lifecycle methods, and constructors.

**CORRECT approach**:
```typescript
function SafeButton() {
  const handleClick = () => {
    try {
      riskyOperation();
    } catch (error) {
      // Handle error locally or use useErrorBoundary from react-error-boundary
      console.error('Click failed:', error);
    }
  };

  return <button onClick={handleClick}>Click Me</button>;
}

// Or with react-error-boundary's useErrorBoundary hook:
function SafeButton() {
  const { showBoundary } = useErrorBoundary();

  const handleClick = () => {
    try {
      riskyOperation();
    } catch (error) {
      showBoundary(error);  // Now the boundary WILL catch it
    }
  };

  return <button onClick={handleClick}>Click Me</button>;
}
```

---

## Anti-Pattern 3: Missing getDerivedStateFromError

```typescript
// WRONG: Only componentDidCatch, no fallback UI
class IncompleteErrorBoundary extends Component<Props, State> {
  state = { hasError: false };

  componentDidCatch(error: Error, info: ErrorInfo): void {
    console.error(error);
    this.setState({ hasError: true });  // Setting state here is unreliable
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

**WHY this is wrong**: `componentDidCatch` runs in the commit phase, AFTER rendering. Without `getDerivedStateFromError`, React attempts to render the broken child tree first, which can cause additional errors or a brief flash of broken UI. ALWAYS use `getDerivedStateFromError` to set the error state — it runs during the render phase and prevents the broken tree from ever being committed.

---

## Anti-Pattern 4: Swallowing Errors Without Logging

```typescript
// WRONG: Error is caught but never reported
class SilentBoundary extends Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  // Missing componentDidCatch — error is silently swallowed
  // No logging, no reporting, no way to diagnose issues

  render() {
    if (this.state.hasError) return <p>Something went wrong.</p>;
    return this.props.children;
  }
}
```

**WHY this is wrong**: Without `componentDidCatch` or an `onError` callback, errors disappear into a black hole. You have no visibility into production failures. ALWAYS log errors to a reporting service in `componentDidCatch`.

---

## Anti-Pattern 5: Wrapping Every Component in a Boundary

```typescript
// WRONG: Excessive granularity
function App() {
  return (
    <ErrorBoundary fallback={<p>Error</p>}>
      <ErrorBoundary fallback={<p>Error</p>}>
        <Header />
      </ErrorBoundary>
      <ErrorBoundary fallback={<p>Error</p>}>
        <Nav />
      </ErrorBoundary>
      <ErrorBoundary fallback={<p>Error</p>}>
        <ErrorBoundary fallback={<p>Error</p>}>
          <Title />
        </ErrorBoundary>
        <ErrorBoundary fallback={<p>Error</p>}>
          <Subtitle />
        </ErrorBoundary>
      </ErrorBoundary>
    </ErrorBoundary>
  );
}
```

**WHY this is wrong**: Excessive boundaries add React tree overhead, make the component tree unreadable, and fragment the fallback UI into tiny pieces that confuse users. Place boundaries at natural isolation points: app root, feature sections, and components known to be error-prone.

---

## Anti-Pattern 6: Error Boundary Without Recovery Path

```typescript
// WRONG: Dead-end error screen with no way out
class DeadEndBoundary extends Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    console.error(error);
  }

  render() {
    if (this.state.hasError) {
      return <p>Something went wrong.</p>;  // No button, no link, no escape
    }
    return this.props.children;
  }
}
```

**WHY this is wrong**: The user is stuck on a dead-end screen. ALWAYS provide at least one recovery option: a "Try Again" button, a link to the home page, or a page refresh button.

**CORRECT approach**:
```typescript
render() {
  if (this.state.hasError) {
    return (
      <div role="alert">
        <p>Something went wrong.</p>
        <button onClick={() => this.setState({ hasError: false, error: null })}>
          Try Again
        </button>
        <a href="/">Return to Home</a>
      </div>
    );
  }
  return this.props.children;
}
```

---

## Anti-Pattern 7: Relying on Error Boundaries for Async Errors

```typescript
// WRONG: Async errors bypass error boundaries
function AsyncComponent() {
  useEffect(() => {
    // This error is NOT caught by any error boundary
    fetch('/api/data')
      .then((res) => res.json())
      .then((data) => {
        throw new Error('Processing failed');  // NOT caught
      });
  }, []);

  return <p>Loading...</p>;
}
```

**WHY this is wrong**: Promise rejections and errors in async code execute outside the React render cycle. Error boundaries cannot intercept them.

**CORRECT approach**:
```typescript
function AsyncComponent() {
  const { showBoundary } = useErrorBoundary();

  useEffect(() => {
    let cancelled = false;
    fetch('/api/data')
      .then((res) => res.json())
      .then((data) => {
        if (!cancelled) processData(data);
      })
      .catch((error) => {
        if (!cancelled) showBoundary(error);  // Bridge to error boundary
      });

    return () => { cancelled = true; };
  }, [showBoundary]);

  return <p>Loading...</p>;
}
```

---

## Anti-Pattern 8: Throwing in getDerivedStateFromError

```typescript
// WRONG: Side effects in getDerivedStateFromError
class BadBoundary extends Component<Props, State> {
  static getDerivedStateFromError(error: Error): State {
    // WRONG: This runs during the render phase — no side effects!
    fetch('/api/log-error', {
      method: 'POST',
      body: JSON.stringify({ error: error.message }),
    });
    return { hasError: true };
  }
}
```

**WHY this is wrong**: `getDerivedStateFromError` runs during the render phase, which must be pure with no side effects. Network requests, DOM manipulation, and other side effects MUST go in `componentDidCatch` (commit phase).

---

## Anti-Pattern 9: Using Error Boundaries for Control Flow

```typescript
// WRONG: Using errors as a data-passing mechanism
function DataComponent({ data }: { data: unknown }) {
  if (!data) {
    throw new Error('NO_DATA');  // Abuse of error boundaries
  }
  return <p>{JSON.stringify(data)}</p>;
}

// Parent checks error type to decide what to show
class ControlFlowBoundary extends Component<Props, State> {
  static getDerivedStateFromError(error: Error): State {
    if (error.message === 'NO_DATA') {
      return { hasError: true, showEmpty: true };
    }
    return { hasError: true, showEmpty: false };
  }
}
```

**WHY this is wrong**: Error boundaries are for unexpected failures, not expected application states. Use conditional rendering for expected states (loading, empty, error). Use error boundaries only as a safety net for genuinely unexpected exceptions.

**CORRECT approach**:
```typescript
function DataComponent({ data }: { data: unknown }) {
  if (!data) return <EmptyState />;
  return <p>{JSON.stringify(data)}</p>;
}
```

---

## Summary: Error Boundary Rules

| Rule | Rationale |
|------|-----------|
| ALWAYS use a class component or `react-error-boundary` | No hook equivalent exists |
| ALWAYS implement both `getDerivedStateFromError` AND `componentDidCatch` | Fallback UI + error logging |
| ALWAYS provide a recovery path in fallback UI | Dead-end screens frustrate users |
| ALWAYS log errors in `componentDidCatch`, not `getDerivedStateFromError` | Render phase must be pure |
| NEVER rely on boundaries for event handler errors | Use `try/catch` or `useErrorBoundary` |
| NEVER rely on boundaries for async errors | Use `.catch()` or `useErrorBoundary` |
| NEVER wrap every component in a boundary | Place at natural isolation points |
| NEVER use error boundaries for control flow | Use conditional rendering instead |
