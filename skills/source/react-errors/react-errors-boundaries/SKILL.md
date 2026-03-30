---
name: react-errors-boundaries
description: >
  Use when building React components that need error recovery and graceful
  failure handling. Prevents the common mistake of letting uncaught errors
  crash the entire application tree. Covers Error Boundary class components,
  getDerivedStateFromError, componentDidCatch, recovery patterns, nested
  boundaries, placement strategy, error boundary limitations.
  Keywords: ErrorBoundary, getDerivedStateFromError, componentDidCatch, fallback UI, app crashes, white screen of death, catch error in component, fallback UI, error recovery..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-errors-boundaries

## Quick Reference

### Error Boundary Overview

| Aspect | Detail |
|--------|--------|
| Component type | Class component ONLY (no hook equivalent exists) |
| Catches | Errors during rendering, lifecycle methods, constructors of child tree |
| Does NOT catch | Event handlers, async code, SSR errors, errors in the boundary itself |
| React 18 | `getDerivedStateFromError` + `componentDidCatch` |
| React 19 | Same API + `onCaughtError`/`onUncaughtError` callbacks on `createRoot` |
| Recommended library | `react-error-boundary` (function component wrapper with hooks) |

### Lifecycle Methods

| Method | Phase | Purpose | Side Effects? |
|--------|-------|---------|---------------|
| `static getDerivedStateFromError(error)` | Render | Return state update to show fallback UI | NO — must be pure |
| `componentDidCatch(error, info)` | Commit | Log errors to reporting service | YES — side effects allowed |

### Critical Warnings

**NEVER** use a function component as an error boundary — React has NO hook equivalent for `getDerivedStateFromError`. Error boundaries MUST be class components or use the `react-error-boundary` library.

**NEVER** rely on error boundaries to catch event handler errors — ALWAYS use `try/catch` inside event handlers. Error boundaries only catch errors during rendering and lifecycle methods.

**NEVER** rely on error boundaries to catch async errors (promises, `setTimeout`) — ALWAYS use `.catch()` or `try/catch` in async code. The one exception: errors thrown inside `startTransition` callbacks ARE caught by error boundaries.

**NEVER** expect an error boundary to catch its own errors — if the boundary itself throws, a PARENT boundary must catch it. ALWAYS have a top-level boundary as a safety net.

**NEVER** use error boundaries as a substitute for proper input validation — error boundaries are a safety net for unexpected failures, not a control flow mechanism.

**ALWAYS** implement both `getDerivedStateFromError` AND `componentDidCatch` — the first renders fallback UI, the second logs the error. Using only one gives an incomplete error boundary.

**ALWAYS** provide a way to recover from errors — a "Try Again" button, navigation link, or key-based reset. A dead-end error screen frustrates users.

---

## Complete TypeScript Error Boundary

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    // Render phase — NO side effects. Return state for fallback UI.
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    // Commit phase — side effects allowed. Log to error service.
    console.error('Error boundary caught:', error);
    console.error('Component stack:', info.componentStack);
    this.props.onError?.(error, info);
  }

  private handleReset = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      if (typeof this.props.fallback === 'function') {
        return this.props.fallback(this.state.error, this.handleReset);
      }
      return this.props.fallback;
    }
    return this.props.children;
  }
}

export default ErrorBoundary;
```

**Usage**:
```typescript
// Static fallback
<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <UserProfile userId={userId} />
</ErrorBoundary>

// Dynamic fallback with reset
<ErrorBoundary
  fallback={(error, reset) => (
    <div role="alert">
      <p>Error: {error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  )}
  onError={(error, info) => logToService(error, info)}
>
  <Dashboard />
</ErrorBoundary>
```

---

## What Error Boundaries DO NOT Catch

| Scenario | Why Not Caught | Use Instead |
|----------|---------------|-------------|
| Event handlers | Run outside React render cycle | `try/catch` in the handler |
| Async code (`setTimeout`, promises) | Executes after render completes | `.catch()` or `try/catch` |
| Server-side rendering (SSR) | Different execution context | Server-side error handling |
| Errors in the boundary itself | Cannot catch its own errors | A parent error boundary |

**Exception**: Errors thrown inside `startTransition` callbacks ARE caught by error boundaries.

---

## Boundary Placement Strategy

### Decision Tree

```
Is this the app root?
├── YES → Page-level boundary (catch-all safety net)
│         Shows generic "Something went wrong" with reload option
│
└── NO → Is this an independent feature/widget?
    ├── YES → Feature-level boundary
    │         Isolates feature failure from rest of page
    │
    └── NO → Is this a single component that might fail?
        ├── YES → Component-level boundary
        │         Granular recovery, minimal UI disruption
        │
        └── NO → No boundary needed here
```

### Placement Levels

```typescript
// Level 1: Page-level (root safety net) — ALWAYS have this
<ErrorBoundary fallback={<FullPageError />}>
  <App />
</ErrorBoundary>

// Level 2: Feature-level (isolate independent sections)
<ErrorBoundary fallback={<SidebarError />}>
  <Sidebar />
</ErrorBoundary>
<ErrorBoundary fallback={<FeedError />}>
  <NewsFeed />
</ErrorBoundary>

// Level 3: Component-level (granular recovery)
<ErrorBoundary fallback={<ChartPlaceholder />}>
  <RevenueChart data={data} />
</ErrorBoundary>
```

**ALWAYS** have at minimum a page-level error boundary wrapping the entire application.

**NEVER** wrap every single component in a boundary — this creates excessive overhead and fragments the UI. Use boundaries at natural isolation points.

---

## Error Recovery Patterns

### Pattern 1: Key Prop Reset

Force React to unmount and remount the component by changing the key:

```typescript
function RecoverableWidget({ userId }: { userId: string }) {
  const [errorKey, setErrorKey] = useState<number>(0);

  return (
    <ErrorBoundary
      key={errorKey}
      fallback={
        <button onClick={() => setErrorKey((k) => k + 1)}>
          Retry
        </button>
      }
    >
      <UserWidget userId={userId} />
    </ErrorBoundary>
  );
}
```

### Pattern 2: Reset Callback

Use the boundary's internal reset method (as shown in the complete implementation above):

```typescript
<ErrorBoundary
  fallback={(error, reset) => (
    <div role="alert">
      <p>Failed to load: {error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  )}
>
  <DataTable />
</ErrorBoundary>
```

### Pattern 3: Navigate Away

Redirect to a safe route on error:

```typescript
function NavigatingFallback({ error }: { error: Error }) {
  return (
    <div role="alert">
      <p>This page encountered an error.</p>
      <a href="/">Return to Home</a>
    </div>
  );
}
```

---

## Nested Boundaries

Inner boundaries catch errors first. If the inner boundary itself fails, the outer boundary catches it.

```typescript
<ErrorBoundary fallback={<AppCrashScreen />}>       {/* Outer: last resort */}
  <Header />
  <ErrorBoundary fallback={<ContentError />}>        {/* Inner: feature-level */}
    <ErrorBoundary fallback={<WidgetError />}>       {/* Innermost: component */}
      <ComplexWidget />
    </ErrorBoundary>
    <OtherContent />
  </ErrorBoundary>
  <Footer />
</ErrorBoundary>
```

**Behavior**: If `ComplexWidget` throws, the innermost boundary shows `<WidgetError />`. The rest of the page (`Header`, `OtherContent`, `Footer`) remains functional.

---

## react-error-boundary Library

The `react-error-boundary` package provides a function-component-based API. ALWAYS use this library instead of writing custom class components.

```bash
npm install react-error-boundary
```

### ErrorBoundary Component

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({
  error,
  resetErrorBoundary,
}: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <div role="alert">
      <p>Something went wrong:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => logToService(error, info)}
  onReset={(details) => {
    // Reset app state that caused the error
  }}
  resetKeys={[userId]}  // Auto-reset when these values change
>
  <UserProfile userId={userId} />
</ErrorBoundary>
```

### useErrorBoundary Hook

Trigger the nearest error boundary from event handlers or async code:

```typescript
import { useErrorBoundary } from 'react-error-boundary';

function UserActions() {
  const { showBoundary } = useErrorBoundary();

  async function handleDelete() {
    try {
      await deleteUser();
    } catch (error) {
      showBoundary(error);  // Triggers nearest ErrorBoundary
    }
  }

  return <button onClick={handleDelete}>Delete Account</button>;
}
```

This solves the limitation that error boundaries cannot catch event handler or async errors.

---

## React 19: Root-Level Error Callbacks

React 19 adds error reporting callbacks on `createRoot`:

```typescript
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root')!, {
  onCaughtError: (error, errorInfo) => {
    // Errors caught by an Error Boundary
    console.error('Caught by boundary:', error, errorInfo.componentStack);
  },
  onUncaughtError: (error, errorInfo) => {
    // Errors NOT caught by any Error Boundary
    console.error('Uncaught:', error, errorInfo.componentStack);
  },
  onRecoverableError: (error, errorInfo) => {
    // Errors React recovers from automatically
    console.error('Recovered:', error, errorInfo.componentStack);
  },
});
```

These callbacks complement error boundaries — they do NOT replace them.

---

## Testing Error Boundaries

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Component that throws on demand
function ThrowingComponent({ shouldThrow }: { shouldThrow: boolean }) {
  if (shouldThrow) {
    throw new Error('Test error');
  }
  return <p>Content is fine</p>;
}

describe('ErrorBoundary', () => {
  // Suppress console.error noise in test output
  const originalError = console.error;
  beforeAll(() => { console.error = vi.fn(); });
  afterAll(() => { console.error = originalError; });

  it('renders children when no error', () => {
    render(
      <ErrorBoundary fallback={<p>Error occurred</p>}>
        <ThrowingComponent shouldThrow={false} />
      </ErrorBoundary>
    );
    expect(screen.getByText('Content is fine')).toBeInTheDocument();
  });

  it('renders fallback when child throws', () => {
    render(
      <ErrorBoundary fallback={<p>Error occurred</p>}>
        <ThrowingComponent shouldThrow={true} />
      </ErrorBoundary>
    );
    expect(screen.getByText('Error occurred')).toBeInTheDocument();
  });

  it('calls onError callback with error details', () => {
    const onError = vi.fn();
    render(
      <ErrorBoundary fallback={<p>Error</p>} onError={onError}>
        <ThrowingComponent shouldThrow={true} />
      </ErrorBoundary>
    );
    expect(onError).toHaveBeenCalledWith(
      expect.objectContaining({ message: 'Test error' }),
      expect.objectContaining({ componentStack: expect.any(String) })
    );
  });
});
```

---

## Development vs Production Behavior

| Behavior | Development | Production |
|----------|-------------|------------|
| Error bubbling | Errors bubble to `window.onerror` even when caught | Only caught by error boundaries |
| Console output | React logs full component stack | Minimal logging |
| Strict Mode | Components render twice to detect impurities | Single render |

---

## Reference Links

- [references/examples.md](references/examples.md) -- Error boundary patterns and real-world implementations
- [references/anti-patterns.md](references/anti-patterns.md) -- Common error boundary mistakes and how to avoid them

### Official Sources

- https://react.dev/reference/react/Component (getDerivedStateFromError, componentDidCatch)
- https://react.dev/reference/react-dom/client/createRoot (onCaughtError, onUncaughtError)
- https://github.com/bvaughn/react-error-boundary (react-error-boundary library)
