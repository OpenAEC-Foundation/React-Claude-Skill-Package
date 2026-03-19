---
name: react-errors-hydration
description: "Diagnoses and resolves React hydration mismatch errors including server/client HTML differences, common causes (Date, Math.random, browser APIs, conditional rendering), suppressHydrationWarning usage, debugging hydration errors, and React 19 hydration improvements. Activates when encountering hydration warnings, debugging SSR/SSG mismatches, or implementing server-rendered React applications."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-errors-hydration

## Quick Reference

### What Hydration Is

Hydration is the process where React attaches event handlers and component state to server-rendered HTML. React expects the server-rendered DOM to EXACTLY match what the client would render on its first pass. Any difference triggers a hydration mismatch error.

```
Server renders HTML string ──> Browser displays HTML (fast, non-interactive)
                                       │
React hydrates ────────────────> Attaches event handlers, state, effects
                                       │
                                Interactive application
```

**ALWAYS** ensure server and client render identical output on the first render pass. Hydration does NOT patch differences -- it assumes the DOM is correct and only attaches interactivity.

**NEVER** treat hydration warnings as harmless. In React 18, mismatches silently produce broken UI. In React 19, React attempts recovery but at a performance cost.

---

## Hydration Mismatch Diagnostic Table

| Error Message / Symptom | Cause | Fix |
|--------------------------|-------|-----|
| "Text content does not match" | Different text on server vs client (Date, Math.random, locale) | Use `useEffect` + state for dynamic values |
| "Expected server HTML to contain a matching `<div>` in `<p>`" | Invalid HTML nesting (`<div>` inside `<p>`, `<p>` inside `<p>`) | Fix HTML nesting to follow spec |
| "Hydration failed because the server-rendered HTML didn't match the client" | Conditional rendering based on client-only state | Use `useEffect` for client-only branches |
| "There was an error while hydrating but React was able to recover" (React 19) | Any mismatch -- React 19 reports and recovers | Fix root cause; recovery re-renders entire tree |
| Content flickers on page load | Mismatch causes React to discard server HTML and re-render | Identify and fix the mismatch source |
| Event handlers not working on server-rendered content | Hydration failed silently (React 18) | Check console for hydration warnings |
| Extra attributes like `data-*` from browser extensions | Extensions inject attributes after server render | Ignore if confirmed extension-caused; see Browser Extensions section |
| "Prop `className` did not match" | CSS-in-JS generating different class names server vs client | Configure SSR for your CSS-in-JS library |

---

## Common Causes (Ranked by Frequency)

### 1. Date/Time Rendering

**Problem**: Server and client run at different times or timezones.

```tsx
// BAD: Hydration mismatch -- server time !== client time
function Header(): JSX.Element {
  return <span>{new Date().toLocaleTimeString()}</span>;
}

// GOOD: Render on client only via useEffect
function Header(): JSX.Element {
  const [time, setTime] = useState<string>("");

  useEffect(() => {
    setTime(new Date().toLocaleTimeString());
  }, []);

  return <span>{time}</span>;
}
```

### 2. Browser-Only APIs (window, localStorage, navigator)

**Problem**: These APIs do not exist on the server.

```tsx
// BAD: window is undefined on server
function Layout(): JSX.Element {
  const width = window.innerWidth;
  return <div>{width > 768 ? <Desktop /> : <Mobile />}</div>;
}

// GOOD: Detect client with useEffect
function Layout(): JSX.Element {
  const [isClient, setIsClient] = useState<boolean>(false);

  useEffect(() => {
    setIsClient(true);
  }, []);

  if (!isClient) {
    return <div><Desktop /></div>; // Server default
  }

  return (
    <div>{window.innerWidth > 768 ? <Desktop /> : <Mobile />}</div>
  );
}
```

### 3. Conditional Rendering Based on Client State

**Problem**: Authentication status, feature flags, or user preferences differ between server and client.

```tsx
// BAD: isLoggedIn differs on server vs client
function Nav(): JSX.Element {
  const isLoggedIn = checkAuth(); // Returns false on server, true on client
  return isLoggedIn ? <UserMenu /> : <LoginButton />;
}

// GOOD: Start with server value, update on client
function Nav(): JSX.Element {
  const [isLoggedIn, setIsLoggedIn] = useState<boolean>(false);

  useEffect(() => {
    setIsLoggedIn(checkAuth());
  }, []);

  return isLoggedIn ? <UserMenu /> : <LoginButton />;
}
```

### 4. Math.random and Non-Deterministic Values

```tsx
// BAD: Different random value on server vs client
function Banner(): JSX.Element {
  const id = Math.random().toString(36).slice(2);
  return <div id={id}>Welcome</div>;
}

// GOOD: Use useId (React 18+) for stable identifiers
function Banner(): JSX.Element {
  const id = useId();
  return <div id={id}>Welcome</div>;
}
```

### 5. HTML Nesting Violations

**ALWAYS** follow HTML nesting rules. The browser auto-corrects invalid nesting BEFORE React hydrates, creating a DOM that does not match React's expected tree.

| Invalid Nesting | Browser Correction | Result |
|----------------|-------------------|--------|
| `<p><div>text</div></p>` | Splits into `<p></p><div>text</div><p></p>` | Mismatch |
| `<a><a>link</a></a>` | Closes outer `<a>` before inner | Mismatch |
| `<table><div>row</div></table>` | Removes `<div>` | Mismatch |
| `<ul><div><li>item</li></div></ul>` | Restructures children | Mismatch |

### 6. CSS-in-JS Class Name Mismatch

CSS-in-JS libraries (styled-components, Emotion) generate class names at runtime. If the server and client use different generation strategies or ordering, class names differ.

**ALWAYS** configure your CSS-in-JS library for SSR following its official documentation. For styled-components, use `ServerStyleSheet`. For Emotion, use `extractCriticalToChunks`.

---

## The isClient Pattern

The standard pattern for client-only rendering:

```tsx
function useIsClient(): boolean {
  const [isClient, setIsClient] = useState<boolean>(false);

  useEffect(() => {
    setIsClient(true);
  }, []);

  return isClient;
}

// Usage
function ClientOnlyFeature(): JSX.Element {
  const isClient = useIsClient();

  if (!isClient) {
    return <Placeholder />; // MUST match server output
  }

  return <RichInteractiveWidget />;
}
```

**NEVER** use `typeof window !== "undefined"` as a render condition -- this evaluates to `true` during client-side rendering of SSR apps (hydration pass), causing the mismatch you are trying to avoid.

---

## suppressHydrationWarning

```tsx
// Acceptable: Timestamps that intentionally differ
<time suppressHydrationWarning>
  {new Date().toISOString()}
</time>

// Acceptable: Third-party widget containers
<div suppressHydrationWarning id="third-party-widget" />
```

**ALWAYS** use `suppressHydrationWarning` ONLY on individual elements where the mismatch is intentional and harmless.

**NEVER** use `suppressHydrationWarning` as a blanket fix on parent containers. It only suppresses one level deep and masks real bugs.

**NEVER** use `suppressHydrationWarning` to hide mismatches you do not understand. Diagnose the root cause first.

---

## hydrateRoot API

```tsx
import { hydrateRoot } from "react-dom/client";
import App from "./App";

const root = hydrateRoot(
  document.getElementById("root") as HTMLElement,
  <App />,
  {
    onRecoverableError(error: unknown, errorInfo: { componentStack?: string }) {
      // Log hydration mismatches to your error tracking service
      console.error("Hydration error:", error);
      console.error("Component stack:", errorInfo.componentStack);
    },
  }
);
```

**ALWAYS** provide `onRecoverableError` in production to track hydration issues. React 19 uses this callback for all recovered hydration mismatches.

---

## React 19 Hydration Improvements

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Error messages | Generic "did not match" text | Full HTML diff showing server vs client output |
| Recovery | Discards entire server-rendered tree on mismatch | Attempts granular recovery, re-renders only affected subtree |
| Reporting | Console warning only | `onRecoverableError` callback with component stack |
| Third-party script interference | Silent failures | Better tolerance for extra attributes from extensions |
| `<style>` and `<link>` in `<head>` | Manual hoisting required | Native support; React deduplicates and hoists automatically |

### React 19 Diff Output Example

```
Warning: Text content did not match.
  Server: "Hello, World"
  Client: "Hello, User"

    at Greeting (app/components/Greeting.tsx:5:3)
    at Layout (app/layout.tsx:12:5)
```

React 19 shows the EXACT server value vs client value plus the full component stack. Use this to trace the mismatch source directly.

---

## Browser Extension Interference

Browser extensions (ad blockers, password managers, translation tools, accessibility plugins) inject or modify DOM elements AFTER server render but BEFORE React hydration.

### Symptoms
- Hydration errors in production that are NOT reproducible locally
- Extra `<div>`, `<style>`, or `data-*` attributes in the DOM
- Errors disappear in incognito mode

### Diagnosis
1. Open the page in incognito mode (extensions disabled)
2. If the error disappears, an extension is the cause
3. Use `onRecoverableError` to log and filter these in production

### Mitigation
- **NEVER** restructure your app to work around extension interference
- **ALWAYS** use `onRecoverableError` to detect and filter extension-caused errors
- Consider wrapping known injection targets with `suppressHydrationWarning` ONLY if confirmed extension-caused

---

## Decision Tree: Fixing Hydration Errors

```
Hydration error detected
├── Is the content time-dependent (Date, timestamp)?
│   └── YES → Use useEffect + state OR suppressHydrationWarning
├── Does it use browser APIs (window, localStorage, navigator)?
│   └── YES → Use the isClient pattern or dynamic import with ssr: false
├── Is it conditional on user/auth state?
│   └── YES → Default to logged-out on server, update in useEffect
├── Is it an HTML nesting violation?
│   └── YES → Fix the HTML structure (no <div> in <p>, etc.)
├── Is it CSS-in-JS class names?
│   └── YES → Configure SSR for your CSS-in-JS library
├── Does it only happen in production with real users?
│   └── YES → Likely browser extension interference; use onRecoverableError
└── None of the above?
    └── Check for: different environment variables, different API responses,
        third-party scripts loading before hydration
```

---

## Dynamic Import for Client-Only Components

When a component fundamentally cannot render on the server:

```tsx
// Next.js
import dynamic from "next/dynamic";

const MapWidget = dynamic(() => import("./MapWidget"), {
  ssr: false,
  loading: () => <div>Loading map...</div>,
});

// Generic React with React.lazy (client-only, not for SSR)
const MapWidget = React.lazy(() => import("./MapWidget"));
```

**ALWAYS** provide a `loading` fallback that matches the server-rendered placeholder to prevent layout shift.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Hydration error examples with complete fix patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Common hydration mistakes and why they fail

### Official Sources

- https://react.dev/reference/react-dom/client/hydrateRoot
- https://react.dev/link/hydration-mismatch
- https://react.dev/reference/react/useId
- https://react.dev/reference/react-dom/client/hydrateRoot#handling-different-client-and-server-content
