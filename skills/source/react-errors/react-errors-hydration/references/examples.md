# Hydration Error Examples with Fixes

## Example 1: User Locale Formatting

### Problem
Server renders with `en-US` locale, client has `de-DE`.

```tsx
// BAD: Locale differs between server and client
function Price({ amount }: { amount: number }): JSX.Element {
  return <span>{amount.toLocaleString()}</span>;
  // Server: "1,000.00" | Client: "1.000,00"
}
```

### Fix
```tsx
function Price({ amount }: { amount: number }): JSX.Element {
  const [formatted, setFormatted] = useState<string>(
    amount.toFixed(2) // Deterministic server fallback
  );

  useEffect(() => {
    setFormatted(amount.toLocaleString());
  }, [amount]);

  return <span>{formatted}</span>;
}
```

---

## Example 2: localStorage-Based Theme

### Problem
Server does not have access to localStorage.

```tsx
// BAD: localStorage is undefined on server
function ThemeProvider({ children }: { children: React.ReactNode }): JSX.Element {
  const theme = localStorage.getItem("theme") || "light";
  return <div className={theme}>{children}</div>;
}
```

### Fix
```tsx
function ThemeProvider({ children }: { children: React.ReactNode }): JSX.Element {
  const [theme, setTheme] = useState<string>("light"); // Server default

  useEffect(() => {
    const saved = localStorage.getItem("theme");
    if (saved) {
      setTheme(saved);
    }
  }, []);

  return <div className={theme}>{children}</div>;
}
```

---

## Example 3: Window Dimensions for Responsive Layout

### Problem
Window dimensions do not exist on the server.

```tsx
// BAD: window is undefined during SSR
function ResponsiveGrid({ children }: { children: React.ReactNode }): JSX.Element {
  const cols = window.innerWidth > 1024 ? 3 : 1;
  return <div style={{ gridTemplateColumns: `repeat(${cols}, 1fr)` }}>{children}</div>;
}
```

### Fix
```tsx
function useWindowWidth(fallback: number = 1024): number {
  const [width, setWidth] = useState<number>(fallback);

  useEffect(() => {
    const handleResize = (): void => setWidth(window.innerWidth);
    handleResize();
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return width;
}

function ResponsiveGrid({ children }: { children: React.ReactNode }): JSX.Element {
  const width = useWindowWidth();
  const cols = width > 1024 ? 3 : 1;
  return <div style={{ gridTemplateColumns: `repeat(${cols}, 1fr)` }}>{children}</div>;
}
```

**Note**: The fallback value (1024) MUST produce the same layout as the server render. Choose a sensible default that matches your server-side assumption.

---

## Example 4: Authentication-Dependent Navigation

### Problem
Auth token exists in cookies on client but is not available during server render in the same way.

```tsx
// BAD: Different auth state between server and client
function NavBar(): JSX.Element {
  const user = getClientSideUser(); // null on server, User on client
  return (
    <nav>
      {user ? (
        <span>Welcome, {user.name}</span>
      ) : (
        <a href="/login">Sign in</a>
      )}
    </nav>
  );
}
```

### Fix
```tsx
function NavBar(): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const currentUser = getClientSideUser();
    setUser(currentUser);
  }, []);

  return (
    <nav>
      {user ? (
        <span>Welcome, {user.name}</span>
      ) : (
        <a href="/login">Sign in</a>
      )}
    </nav>
  );
}
```

**ALWAYS** default to the unauthenticated state on the server. The flash of unauthenticated content is preferable to a hydration mismatch.

---

## Example 5: useId for Stable Identifiers

### Problem
Dynamically generated IDs differ between server and client.

```tsx
// BAD: Math.random produces different values
function FormField({ label }: { label: string }): JSX.Element {
  const id = `field-${Math.random().toString(36).slice(2)}`;
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

### Fix
```tsx
import { useId } from "react";

function FormField({ label }: { label: string }): JSX.Element {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

`useId` generates the same ID on server and client. It is designed specifically for this purpose. Available in React 18+.

---

## Example 6: HTML Nesting Violation

### Problem
Browser auto-corrects invalid HTML before React hydrates.

```tsx
// BAD: <div> is not valid inside <p>
function Article({ content }: { content: string }): JSX.Element {
  return (
    <p>
      <div className="highlight">{content}</div>
    </p>
  );
}
```

### Fix
```tsx
// GOOD: Use <span> inside <p>, or <div> wrapping <p>
function Article({ content }: { content: string }): JSX.Element {
  return (
    <p>
      <span className="highlight">{content}</span>
    </p>
  );
}

// Or restructure:
function Article({ content }: { content: string }): JSX.Element {
  return (
    <div>
      <div className="highlight">{content}</div>
    </div>
  );
}
```

---

## Example 7: Dynamic Import (Next.js) for Client-Only Components

### Problem
A map library requires `window` and cannot render on the server.

```tsx
// BAD: MapContainer accesses window internally
import { MapContainer, TileLayer } from "react-leaflet";

function LocationMap(): JSX.Element {
  return (
    <MapContainer center={[51.505, -0.09]} zoom={13}>
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
    </MapContainer>
  );
}
```

### Fix (Next.js)
```tsx
import dynamic from "next/dynamic";

const LocationMap = dynamic(
  () => import("../components/LocationMap"),
  {
    ssr: false,
    loading: () => (
      <div style={{ height: 400, background: "#eee" }}>Loading map...</div>
    ),
  }
);

// Use normally in pages/components
function Page(): JSX.Element {
  return <LocationMap />;
}
```

---

## Example 8: Timestamps with suppressHydrationWarning

### Problem
Relative timestamps like "3 minutes ago" always differ between server render time and client hydration time.

```tsx
// Acceptable use of suppressHydrationWarning
function Comment({ createdAt }: { createdAt: string }): JSX.Element {
  return (
    <article>
      <p>{comment.text}</p>
      <time suppressHydrationWarning dateTime={createdAt}>
        {formatRelativeTime(createdAt)}
      </time>
    </article>
  );
}
```

This is one of the FEW legitimate uses of `suppressHydrationWarning`. The timestamp will correct itself on the client, and the brief mismatch is acceptable.

---

## Example 9: hydrateRoot with Error Tracking

### Production Setup
```tsx
import { hydrateRoot } from "react-dom/client";
import { reportError } from "./errorTracking";
import App from "./App";

const EXTENSION_PATTERNS = [
  /data-grammarly/,
  /data-lastpass/,
  /data-dashlanecreated/,
];

function isExtensionError(error: unknown): boolean {
  const message = error instanceof Error ? error.message : String(error);
  return EXTENSION_PATTERNS.some((pattern) => pattern.test(message));
}

hydrateRoot(
  document.getElementById("root") as HTMLElement,
  <App />,
  {
    onRecoverableError(error: unknown, errorInfo: { componentStack?: string }) {
      if (isExtensionError(error)) {
        return; // Ignore extension-caused mismatches
      }

      reportError({
        type: "hydration-mismatch",
        error,
        componentStack: errorInfo.componentStack,
      });
    },
  }
);
```

**ALWAYS** filter known extension patterns in `onRecoverableError` to reduce noise in production error tracking.
