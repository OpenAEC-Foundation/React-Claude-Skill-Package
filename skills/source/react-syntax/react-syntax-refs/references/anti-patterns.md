# react-syntax-refs — Anti-Patterns

> Common ref mistakes with explanations and corrections. Every anti-pattern includes WHY it fails and the correct alternative.

---

## AP-001: Reading ref.current During Render

```typescript
// WRONG — breaks component purity
function Display(): JSX.Element {
  const ref = useRef<number>(0);
  ref.current += 1; // Writing during render
  return <p>Renders: {ref.current}</p>; // Reading during render
}
```

**Why it fails**: React may call render functions multiple times (StrictMode, concurrent features, Suspense retries). Reading/writing refs during render produces inconsistent results because React does not track ref mutations.

```typescript
// CORRECT — use state for values that affect render output
function Display(): JSX.Element {
  const [renderCount, setRenderCount] = useState<number>(0);

  useEffect(() => {
    setRenderCount((prev) => prev + 1);
  }, []);

  return <p>Renders: {renderCount}</p>;
}
```

**Rule**: If the value appears in JSX, ALWAYS use `useState`. Refs are for values that do NOT affect rendered output.

---

## AP-002: Using forwardRef in React 19

```typescript
// WRONG in React 19 — forwardRef is deprecated
const MyInput = forwardRef<HTMLInputElement, Props>((props, ref) => {
  return <input ref={ref} {...props} />;
});
```

**Why it fails**: `forwardRef` adds unnecessary complexity in React 19 where `ref` is a regular prop. It still works but is deprecated and adds a wrapper layer.

```typescript
// CORRECT in React 19 — ref as prop
function MyInput({ ref, ...props }: Props & { ref?: Ref<HTMLInputElement> }): JSX.Element {
  return <input ref={ref} {...props} />;
}
```

**Rule**: In React 19 projects, NEVER use `forwardRef`. In React 18 projects or libraries supporting both, `forwardRef` is still required.

---

## AP-003: Exposing Full DOM Node via useImperativeHandle

```typescript
// WRONG — exposes entire DOM API
function CustomInput({ ref }: { ref?: Ref<HTMLInputElement> }): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => inputRef.current!, []);
  return <input ref={inputRef} />;
}
```

**Why it fails**: The parent receives the full `HTMLInputElement` API, breaking encapsulation. The child component cannot change its internal structure without breaking consumers. The non-null assertion (`!`) can cause runtime errors if the ref is not yet attached.

```typescript
// CORRECT — expose only what the parent needs
interface CustomInputHandle {
  focus: () => void;
  clear: () => void;
}

function CustomInput({ ref }: { ref?: Ref<CustomInputHandle> }): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus() { inputRef.current?.focus(); },
    clear() { if (inputRef.current) inputRef.current.value = ""; },
  }), []);

  return <input ref={inputRef} />;
}
```

**Rule**: ALWAYS define a TypeScript interface for the handle. ALWAYS expose specific methods, NEVER the raw node.

---

## AP-004: Creating Callback Ref Inline Without Stability

```typescript
// WRONG — new function every render, causes detach/reattach cycle
function Measurer(): JSX.Element {
  const [height, setHeight] = useState<number>(0);

  return (
    <div
      ref={(node: HTMLDivElement | null) => {
        if (node !== null) {
          setHeight(node.getBoundingClientRect().height);
        }
      }}
    >
      Height: {height}
    </div>
  );
}
```

**Why it fails**: The inline function creates a new reference every render. React detaches the old ref (calls it with `null`) and attaches the new one (calls it with the node) on every render. This causes unnecessary DOM measurements and potential flicker. If `setHeight` triggers a re-render, it creates an infinite loop.

```typescript
// CORRECT — stable callback ref with useCallback
function Measurer(): JSX.Element {
  const [height, setHeight] = useState<number>(0);

  const measuredRef = useCallback((node: HTMLDivElement | null) => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  }, []);

  return (
    <div ref={measuredRef}>Height: {height}</div>
  );
}
```

**Rule**: ALWAYS wrap callback refs in `useCallback` when the callback triggers state updates or performs expensive operations.

---

## AP-005: Using useRef Where useState Is Needed

```typescript
// WRONG — UI never updates
function Counter(): JSX.Element {
  const countRef = useRef<number>(0);

  function increment(): void {
    countRef.current += 1;
    // Component does NOT re-render
  }

  return (
    <div>
      <p>Count: {countRef.current}</p> {/* Always shows 0 */}
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

**Why it fails**: Mutating `ref.current` does not trigger a re-render. The displayed value never updates.

```typescript
// CORRECT — use state for rendered values
function Counter(): JSX.Element {
  const [count, setCount] = useState<number>(0);

  function increment(): void {
    setCount((prev) => prev + 1);
  }

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

**Rule**: If the value appears in JSX output, ALWAYS use `useState`. Use `useRef` ONLY for values that do not affect what is rendered.

---

## AP-006: Forgetting null Check on DOM Ref

```typescript
// WRONG — runtime error if ref not attached yet
function AutoScroll(): JSX.Element {
  const divRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    divRef.current.scrollTop = divRef.current.scrollHeight; // TypeError if null
  }, []);

  return <div ref={divRef}>Content</div>;
}
```

**Why it fails**: DOM refs are `null` until after the component mounts and the ref attaches. Accessing properties on `null` throws a `TypeError`.

```typescript
// CORRECT — always null-check DOM refs
function AutoScroll(): JSX.Element {
  const divRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (divRef.current !== null) {
      divRef.current.scrollTop = divRef.current.scrollHeight;
    }
  }, []);

  return <div ref={divRef}>Content</div>;
}
```

**Rule**: ALWAYS null-check DOM refs before accessing properties. Use optional chaining (`ref.current?.method()`) or explicit null guards.

---

## AP-007: Using String Refs (Legacy)

```typescript
// WRONG — string refs are removed in React 19
class OldComponent extends React.Component {
  render() {
    return <input ref="myInput" />;
  }
}
```

**Why it fails**: String refs were removed in React 19 and have been deprecated since React 16.3. They have performance issues and do not work with function components.

```typescript
// CORRECT — use useRef in function components
function ModernComponent(): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);
  return <input ref={inputRef} />;
}
```

**Rule**: NEVER use string refs. ALWAYS use `useRef` or callback refs.

---

## AP-008: Missing Dependencies in useImperativeHandle

```typescript
// WRONG — stale closure over inputRef
function SearchInput({ ref }: { ref?: Ref<SearchHandle> }): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);
  const [query, setQuery] = useState<string>("");

  useImperativeHandle(ref, () => ({
    getValue() { return query; }, // Captures initial query value
    focus() { inputRef.current?.focus(); },
  }), []); // Missing 'query' dependency

  return <input ref={inputRef} value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

**Why it fails**: The `getValue` method captures the initial value of `query` due to the empty dependency array. The parent always receives the stale initial value.

```typescript
// CORRECT — include dependencies
useImperativeHandle(ref, () => ({
  getValue() { return query; },
  focus() { inputRef.current?.focus(); },
}), [query]); // query included in deps
```

**Rule**: ALWAYS include all reactive values used inside `createHandle` in the dependency array. The linter (`eslint-plugin-react-hooks`) catches this.

---

## AP-009: Returning Cleanup from Callback Ref in React 18

```typescript
// WRONG in React 18 — cleanup return not supported
function TrackedElement(): JSX.Element {
  const trackRef = (node: HTMLDivElement | null): (() => void) => {
    if (node !== null) {
      analytics.trackImpression(node);
      return () => analytics.untrackImpression(node); // Ignored in React 18
    }
    return () => {};
  };

  return <div ref={trackRef}>Tracked</div>;
}
```

**Why it fails**: React 18 does not support cleanup return values from callback refs. The returned function is silently ignored. React 18 passes `null` to the callback on unmount instead.

```typescript
// CORRECT for React 18 — handle null as cleanup signal
function TrackedElement(): JSX.Element {
  const nodeRef = useRef<HTMLDivElement | null>(null);

  const trackRef = useCallback((node: HTMLDivElement | null): void => {
    if (nodeRef.current !== null) {
      analytics.untrackImpression(nodeRef.current); // Cleanup previous
    }
    if (node !== null) {
      analytics.trackImpression(node); // Setup new
    }
    nodeRef.current = node;
  }, []);

  return <div ref={trackRef}>Tracked</div>;
}
```

**Rule**: In React 18, ALWAYS handle cleanup in the `null` branch of callback refs. In React 19, you MAY return a cleanup function instead.
