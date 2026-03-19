# Error Boundary Patterns — Examples

## Pattern 1: Minimal Reusable Error Boundary

The simplest production-ready boundary with TypeScript types:

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(_error: Error): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    console.error('ErrorBoundary caught:', error, info.componentStack);
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

---

## Pattern 2: Error Boundary with Reset Capability

Allows users to recover by retrying the failed operation:

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ResettableErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    this.props.onError?.(error, info);
  }

  private reset = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      return this.props.fallback(this.state.error, this.reset);
    }
    return this.props.children;
  }
}

// Usage
<ResettableErrorBoundary
  fallback={(error, reset) => (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  )}
  onError={(error, info) => {
    errorReportingService.log(error, info.componentStack);
  }}
>
  <Dashboard />
</ResettableErrorBoundary>
```

---

## Pattern 3: Key-Based Recovery

Force full remount by changing the boundary's key:

```typescript
import { useState } from 'react';

function RecoverableSection() {
  const [boundaryKey, setBoundaryKey] = useState<number>(0);

  return (
    <ErrorBoundary
      key={boundaryKey}
      fallback={
        <div role="alert">
          <p>This section failed to load.</p>
          <button onClick={() => setBoundaryKey((k) => k + 1)}>
            Reload Section
          </button>
        </div>
      }
    >
      <ComplexFeature />
    </ErrorBoundary>
  );
}
```

**When to use**: When the error is caused by corrupted component state. Changing the key forces React to destroy the entire subtree and create it fresh.

---

## Pattern 4: Multi-Level Boundary Strategy

A real application with boundaries at three levels:

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    // Level 1: App-wide safety net
    <ErrorBoundary
      FallbackComponent={FullPageError}
      onError={(error, info) => logCritical(error, info)}
    >
      <Header />
      <main>
        {/* Level 2: Feature isolation */}
        <ErrorBoundary
          FallbackComponent={SidebarError}
          resetKeys={[currentRoute]}
        >
          <Sidebar />
        </ErrorBoundary>

        <ErrorBoundary
          FallbackComponent={ContentError}
          resetKeys={[currentRoute]}
        >
          {/* Level 3: Component-level */}
          <ErrorBoundary fallbackRender={({ error }) => (
            <ChartPlaceholder message={error.message} />
          )}>
            <RevenueChart />
          </ErrorBoundary>

          <ErrorBoundary fallbackRender={({ error }) => (
            <TablePlaceholder message={error.message} />
          )}>
            <DataTable />
          </ErrorBoundary>
        </ErrorBoundary>
      </main>
      <Footer />
    </ErrorBoundary>
  );
}

function FullPageError({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <div role="alert" style={{ padding: '2rem', textAlign: 'center' }}>
      <h1>Application Error</h1>
      <p>An unexpected error occurred. Please try refreshing the page.</p>
      <button onClick={resetErrorBoundary}>Refresh</button>
    </div>
  );
}

function SidebarError({ resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <aside>
      <p>Sidebar unavailable</p>
      <button onClick={resetErrorBoundary}>Retry</button>
    </aside>
  );
}
```

---

## Pattern 5: react-error-boundary with resetKeys

Auto-reset the boundary when specific values change (e.g., route navigation):

```typescript
import { ErrorBoundary } from 'react-error-boundary';
import { useLocation } from 'react-router-dom';

function PageContent() {
  const location = useLocation();

  return (
    <ErrorBoundary
      FallbackComponent={PageError}
      resetKeys={[location.pathname]}
      onReset={(details) => {
        // Optionally clear any cached state
        queryClient.clear();
      }}
    >
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </ErrorBoundary>
  );
}
```

**Behavior**: When the user navigates to a different route, `resetKeys` changes, and the error boundary automatically clears its error state. No manual "Try Again" needed.

---

## Pattern 6: useErrorBoundary for Async/Event Errors

Bridge event handler and async errors into the error boundary system:

```typescript
import { useErrorBoundary } from 'react-error-boundary';

function DataLoader({ endpoint }: { endpoint: string }) {
  const { showBoundary } = useErrorBoundary();
  const [data, setData] = useState<unknown>(null);

  useEffect(() => {
    let cancelled = false;
    fetch(endpoint)
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((json) => {
        if (!cancelled) setData(json);
      })
      .catch((error) => {
        if (!cancelled) showBoundary(error);
      });

    return () => { cancelled = true; };
  }, [endpoint, showBoundary]);

  if (!data) return <p>Loading...</p>;
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}

// Wrap in ErrorBoundary
<ErrorBoundary FallbackComponent={DataError}>
  <DataLoader endpoint="/api/users" />
</ErrorBoundary>
```

---

## Pattern 7: Error Boundary with Error Reporting Service

Production pattern with structured error logging:

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface ErrorReport {
  error: {
    name: string;
    message: string;
    stack?: string;
  };
  componentStack: string;
  timestamp: string;
  userAgent: string;
  url: string;
}

interface Props {
  children: ReactNode;
  fallback: ReactNode;
  serviceName: string;
}

interface State {
  hasError: boolean;
}

class ReportingErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(_error: Error): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    const report: ErrorReport = {
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack,
      },
      componentStack: info.componentStack ?? '',
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: window.location.href,
    };

    // Send to your error reporting service
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(report),
    }).catch(() => {
      // Silently fail — do not throw in error handler
    });
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

---

## Pattern 8: React 19 Root-Level Error Callbacks

Complement error boundaries with global error monitoring:

```typescript
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root')!, {
  onCaughtError: (error, errorInfo) => {
    // Fires when an Error Boundary catches an error
    analytics.track('error_caught', {
      message: error.message,
      stack: errorInfo.componentStack,
    });
  },
  onUncaughtError: (error, errorInfo) => {
    // Fires when an error is NOT caught by any boundary
    analytics.track('error_uncaught', {
      message: error.message,
      stack: errorInfo.componentStack,
      severity: 'critical',
    });
    // Show global error overlay
    showGlobalErrorOverlay(error);
  },
  onRecoverableError: (error, errorInfo) => {
    // Fires when React recovers from an error automatically
    // (e.g., hydration mismatch that falls back to client render)
    analytics.track('error_recovered', {
      message: error.message,
    });
  },
});

root.render(<App />);
```
