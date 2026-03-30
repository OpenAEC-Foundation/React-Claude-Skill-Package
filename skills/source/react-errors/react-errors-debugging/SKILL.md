---
name: react-errors-debugging
description: >
  Use when debugging React applications, interpreting console warnings, profiling
  performance, or understanding Strict Mode double-rendering. Prevents the common
  mistake of ignoring React warnings or misdiagnosing Strict Mode behavior as
  bugs. Covers React DevTools, Strict Mode, console warnings, Profiler, component
  stack traces, development vs production error differences.
  Keywords: DevTools, Strict Mode, Profiler, console warnings, stack traces, white screen, blank page, nothing renders, component not showing, React DevTools, double render..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-errors-debugging

## Quick Reference

### Debugging Tools Overview

| Tool | Purpose | When to Use |
|------|---------|-------------|
| React DevTools (Components) | Inspect props, state, hooks, context | Component data inspection |
| React DevTools (Profiler) | Render timing, flamegraph, ranked chart | Performance investigation |
| Browser DevTools (Console) | Read React warnings and error messages | Error diagnosis |
| Browser DevTools (Sources) | Set breakpoints, step through code | Logic debugging |
| `<StrictMode>` | Surface impure renders, unsafe effects | Development-time bug prevention |
| `why-did-you-render` | Log unnecessary re-renders with reasons | Re-render optimization |
| Source maps | Map production errors to original source | Production debugging |

### Critical Warnings

**NEVER** remove `<StrictMode>` to "fix" double-rendering issues -- the double invocation is intentional. It exposes impure components and side effects. ALWAYS fix the underlying impurity instead.

**NEVER** ignore React console warnings -- they indicate real bugs or future breaking changes. ALWAYS resolve every warning before shipping.

**NEVER** debug production builds without source maps -- error messages are stripped and minified in production. ALWAYS generate source maps for staging/debugging environments.

**ALWAYS** check the component stack trace when diagnosing errors -- React prints the component tree path that led to the error, which pinpoints the source.

**ALWAYS** use the Profiler tab (not just the Components tab) when investigating performance -- visual inspection of the component tree does NOT reveal render timing.

---

## Strict Mode

`<StrictMode>` is a development-only tool that helps find bugs by intentionally double-invoking certain functions.

### What Strict Mode Double-Invokes

| Function Type | Double-Invoked? | Why |
|---------------|----------------|-----|
| Component function body | Yes | Detects impure rendering (side effects during render) |
| `useState` initializer | Yes | Detects impure initialization |
| `useReducer` reducer | Yes | Detects impure reducers |
| `useReducer` initializer | Yes | Detects impure initialization |
| `useMemo` callback | Yes | Detects impure memoization |
| `useEffect` setup + cleanup | Yes (mount/unmount/remount) | Detects missing cleanup logic |
| `useRef` — NO | No | Refs are not double-invoked |

### Strict Mode Behavior

```tsx
// StrictMode wraps your app (or a subtree)
import { StrictMode } from "react";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

In development, React will:
1. Render component -> call cleanup -> re-render component (simulating unmount/remount)
2. Call state initializers twice and discard the first result
3. Call reducers twice and discard the first result

### Common Strict Mode Mistakes

```tsx
// BAD: Side effect in render — Strict Mode exposes this by double-calling
function Counter(): JSX.Element {
  const [count, setCount] = useState(0);
  document.title = `Count: ${count}`; // NEVER do this in render
  return <div>{count}</div>;
}

// GOOD: Side effect in useEffect
function Counter(): JSX.Element {
  const [count, setCount] = useState(0);
  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);
  return <div>{count}</div>;
}
```

```tsx
// BAD: Effect without cleanup — Strict Mode exposes the leak
function ChatRoom({ roomId }: { roomId: string }): JSX.Element {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    // Missing cleanup! StrictMode will connect TWICE
  }, [roomId]);
  return <div>Chat: {roomId}</div>;
}

// GOOD: Effect with proper cleanup
function ChatRoom({ roomId }: { roomId: string }): JSX.Element {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect(); // ALWAYS clean up
  }, [roomId]);
  return <div>Chat: {roomId}</div>;
}
```

---

## Console Warnings Diagnostic Table

| Warning Message | Cause | Fix |
|----------------|-------|-----|
| "Each child in a list should have a unique 'key' prop" | Array items rendered without `key` or with duplicate/index keys | ALWAYS provide a stable, unique `key` from data (e.g., `id`). NEVER use array index as key when items reorder. |
| "Cannot update a component (`X`) while rendering a different component (`Y`)" | Calling `setState` of component X inside the render body of component Y | Move the state update into a `useEffect` or event handler. NEVER call setState during render of another component. |
| "Maximum update depth exceeded" | Infinite re-render loop from setState in useEffect without proper deps | Check `useEffect` dependencies — ensure the effect does not unconditionally trigger its own dependency. Add missing deps or restructure logic. |
| "Objects are not valid as a React child" | Passing an object/array directly as JSX child text | Convert to string with `JSON.stringify()`, or map array to elements. NEVER render raw objects. |
| "Function components cannot be given refs" | Passing `ref` to a function component without `forwardRef` | Wrap component with `forwardRef` (React 18) or accept `ref` as a prop (React 19). |
| "Can't perform a React state update on an unmounted component" | setState called after component unmounted (React 17 and earlier) | This warning was REMOVED in React 18. If seen in React 18+, you have a version mismatch. In React 17, use cleanup to cancel async operations. |
| "Invalid hook call" | Hook called outside component/hook, or mismatched React versions | Verify hooks are at top level of component/hook. Check for duplicate React installations with `npm ls react`. |
| "Rendered more hooks than during the previous render" | Conditional hook call — hook count changed between renders | NEVER call hooks inside conditions, loops, or early returns. ALWAYS call all hooks in the same order. |

---

## React DevTools

### Components Tab

The Components tab shows the live component tree with inspectable data:

| Feature | What It Shows | How to Use |
|---------|--------------|------------|
| Component tree | Hierarchy of mounted components | Click to select, use search to filter |
| Props panel | Current props of selected component | Expand objects, edit values live |
| Hooks panel | Current hook values (state, ref, memo, context) | Shows hook name and current value |
| Source link | "< >" icon links to component source | Click to open in Sources tab |
| Rendered by | Which parent rendered this component | Shows in sidebar when component selected |
| Owner stack (React 19) | Full owner component chain | Shows which component created this one |

### Profiler Tab

| Feature | Purpose | Interpretation |
|---------|---------|---------------|
| Flamegraph | Visualize render time per component | Wide bars = slow renders. Grey bars = did not render. |
| Ranked chart | Components sorted by render duration | Focus optimization on top-ranked components |
| "Why did this render?" | Shows render trigger reason | Enable via gear icon -> "Record why each component rendered" |
| Commit selector | Browse individual render commits | Each commit = one React render batch |
| Render duration | Time in ms for each component | Compare before/after optimization |

### Profiler Workflow

1. Open React DevTools -> Profiler tab
2. Click gear icon -> enable "Record why each component rendered while profiling"
3. Click Record (blue circle)
4. Perform the interaction you want to measure
5. Click Stop
6. Analyze: flamegraph for overview, ranked chart for worst offenders
7. Check "Why did this render?" for each slow component

---

## Component Stack Traces

When React throws an error, it prints a **component stack trace** showing the component hierarchy:

```
Error: Something went wrong
    at UserProfile (UserProfile.tsx:15)
    at div
    at Dashboard (Dashboard.tsx:8)
    at ErrorBoundary (ErrorBoundary.tsx:5)
    at App (App.tsx:12)
```

### Reading Component Stacks

- Read **top to bottom**: the topmost component is where the error occurred
- Native HTML elements (e.g., `at div`) appear in the stack
- File names and line numbers point to the component definition
- ALWAYS look at the first custom component in the stack -- that is the error source

### React 19 Improvements

React 19 introduces **owner stacks** via `captureOwnerStack()`:

```tsx
import { captureOwnerStack } from "react";

// In an error boundary or error handler:
function handleError(error: Error): void {
  const ownerStack = captureOwnerStack();
  console.error("Error:", error.message);
  console.error("Owner stack:", ownerStack);
  // Owner stack shows WHICH component created the erroring component
  // (not just the render tree, but the ownership/creation chain)
}
```

---

## Common Error Messages Diagnostic Table

| Error Message | Cause | Fix |
|---------------|-------|-----|
| `Minified React error #XXX` | Production build with stripped error messages | Look up the error code at `https://react.dev/errors/XXX` or debug with development build |
| `Too many re-renders` | Component unconditionally calls setState during render | NEVER call `setState` directly in component body. Use `useEffect` or event handlers. |
| `Hydration failed because the initial UI does not match` | Server HTML differs from client render | Ensure server and client render identical output. Check for `typeof window` guards, `Date.now()`, or random values in render. |
| `Cannot read properties of null (reading 'useState')` | Multiple React copies or hook called outside component | Run `npm ls react` to check for duplicates. Ensure hooks are called inside components only. |
| `Uncaught Invariant Violation` | Internal React assertion failed | Usually indicates a bug in your code violating React rules. Check the message details. |
| `Element type is invalid: expected a string or class/function` | Component is `undefined` at render time | Check import statement — likely a typo, default vs named export mismatch, or circular dependency. |
| `Cannot update during an existing state transition` | `setState` called inside `render()` or `getDerivedStateFromProps` | Move state update to `useEffect` or `componentDidMount`/`componentDidUpdate`. |

---

## Development vs Production Differences

| Aspect | Development | Production |
|--------|-------------|------------|
| Error messages | Full descriptive messages with component stacks | Minified error codes (e.g., `Minified React error #310`) |
| Strict Mode | Active (double-invokes renders, effects) | Completely stripped — no double invocations |
| Console warnings | All warnings displayed | Most warnings suppressed |
| Performance | Slower due to extra checks, Strict Mode overhead | Optimized, no dev-only checks |
| Profiling | DevTools Profiler works out of the box | Requires profiling build: `react-dom/profiling` |
| Source maps | Available by default (dev server) | Must be explicitly generated and deployed |
| PropTypes | Validated at runtime (if used) | Stripped entirely |

### Debugging Production Errors

1. **Look up minified error codes**: Visit `https://react.dev/errors/{code}` for the full message
2. **Generate source maps**: Configure your bundler to produce `.map` files for staging
3. **Use profiling build** for production performance analysis:

```tsx
// webpack.config.ts — alias for profiling build
resolve: {
  alias: {
    "react-dom$": "react-dom/profiling",
    "scheduler/tracing": "scheduler/tracing-profiling",
  },
}
```

4. **Error monitoring**: Use services that can decode source maps server-side (Sentry, Bugsnag)

---

## React 19 Debugging Improvements

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Hydration mismatch errors | Generic "did not match" message | Detailed diff showing exact HTML differences |
| Error reporting | Component stack only | Owner stacks via `captureOwnerStack()` |
| Third-party script handling | Hydration errors on injected elements | Automatically skips unexpected `<script>`, `<style>`, `<link>` in `<head>`/`<body>` |
| ref on function components | Requires `forwardRef` wrapper | `ref` is a regular prop — no wrapper needed |
| useEffect cleanup timing | Synchronous during unmount | Consistent async cleanup with transition support |

---

## Debugging Tools Setup

### React DevTools Extension

ALWAYS install the React DevTools browser extension for Chrome or Firefox. It adds the Components and Profiler tabs to browser DevTools.

- Chrome: Search "React Developer Tools" in Chrome Web Store
- Firefox: Search "React Developer Tools" in Firefox Add-ons
- Standalone (for React Native/iframe): `npx react-devtools`

### why-did-you-render Library

For granular re-render tracking beyond DevTools Profiler:

```tsx
// wdyr.ts — MUST be imported BEFORE React
import React from "react";

if (process.env.NODE_ENV === "development") {
  const whyDidYouRender = await import("@welldone-software/why-did-you-render");
  whyDidYouRender.default(React, {
    trackAllPureComponents: true,
  });
}

// Then on specific components:
MyComponent.whyDidYouRender = true;
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Debugging workflow examples with step-by-step instructions
- [references/anti-patterns.md](references/anti-patterns.md) -- Common debugging mistakes and how to avoid them

### Official Sources

- https://react.dev/learn/react-developer-tools
- https://react.dev/reference/react/StrictMode
- https://react.dev/reference/react/Profiler
- https://react.dev/errors
- https://react.dev/blog/2024/04/25/react-19
