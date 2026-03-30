---
name: react-syntax-refs
description: >
  Use when accessing DOM elements, forwarding refs to child components,
  implementing imperative APIs, or managing mutable values across renders.
  Prevents the common mistake of using refs for values that should trigger
  re-renders. Covers useRef, forwardRef, useImperativeHandle, callback refs,
  React 19 ref cleanup, React 19 ref as prop.
  Keywords: useRef, forwardRef, useImperativeHandle, callback ref, DOM access.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-refs

## Quick Reference

### Ref Hooks and APIs

| API | Purpose | React 18 | React 19 |
|-----|---------|----------|----------|
| `useRef<T>` | Hold mutable value or DOM reference | Yes | Yes |
| `forwardRef` | Pass ref through component | Required | Deprecated (ref is a prop) |
| `useImperativeHandle` | Expose limited API to parent | With `forwardRef` | With ref prop |
| Callback refs | Function-based ref assignment | Yes | Yes + cleanup return |

### TypeScript Signatures

```typescript
// useRef — DOM element
const ref = useRef<HTMLInputElement>(null);

// useRef — mutable value (no null in generic = mutable)
const countRef = useRef<number>(0);

// useImperativeHandle
useImperativeHandle<HandleType>(ref, () => ({ method() {} }), [deps]);
```

### Critical Warnings

**NEVER** read or write `ref.current` during rendering. The only exception is lazy initialization (`if (ref.current === null) ref.current = new Thing()`). Reading/writing refs during render breaks component purity and causes unpredictable behavior.

**NEVER** use `forwardRef` in new React 19 code. Pass `ref` as a regular prop instead. `forwardRef` is deprecated in React 19.

**NEVER** expose the entire DOM node via `useImperativeHandle`. ALWAYS expose only the specific methods the parent needs. Exposing the full node breaks encapsulation.

**ALWAYS** type `useRef` with the correct generic. For DOM refs, ALWAYS initialize with `null` and type as `useRef<HTMLElement>(null)`. For mutable values, initialize with the actual value.

---

## Decision Tree: Ref vs State

```
Need to store a value across renders?
  |
  +-- Does changing this value need to update the UI?
  |     YES --> Use useState or useReducer
  |     NO  --> Use useRef
  |
  +-- Is this a DOM element reference?
  |     YES --> Use useRef<HTMLElement>(null), attach via ref prop
  |
  +-- Is this a timer ID, previous value, or instance variable?
        YES --> Use useRef<T>(initialValue)
```

**Rule**: If the value is used ONLY in event handlers or effects and NEVER in JSX output, use `useRef`. If it appears in rendered output, use `useState`.

---

## useRef: DOM Access

```typescript
import { useRef } from "react";

function TextInput(): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);

  function handleClick(): void {
    inputRef.current?.focus();
  }

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={handleClick}>Focus</button>
    </>
  );
}
```

### TypeScript Generics for DOM Refs

| Element | Generic Type |
|---------|-------------|
| `<input>` | `useRef<HTMLInputElement>(null)` |
| `<div>` | `useRef<HTMLDivElement>(null)` |
| `<canvas>` | `useRef<HTMLCanvasElement>(null)` |
| `<video>` | `useRef<HTMLVideoElement>(null)` |
| `<form>` | `useRef<HTMLFormElement>(null)` |
| `<svg>` | `useRef<SVGSVGElement>(null)` |

ALWAYS initialize DOM refs with `null`. The element is not available until after mount.

---

## useRef: Mutable Values

```typescript
import { useRef, useEffect } from "react";

function Timer(): JSX.Element {
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const renderCountRef = useRef<number>(0);

  useEffect(() => {
    intervalRef.current = setInterval(() => {
      renderCountRef.current += 1;
    }, 1000);

    return () => {
      if (intervalRef.current !== null) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  return <div>Timer running</div>;
}
```

**Key behavior**: Changing `.current` does NOT trigger a re-render. React is not aware of ref mutations. The same object reference persists across every render.

---

## forwardRef (React 18) vs ref Prop (React 19)

### React 18: forwardRef Required

```typescript
import { forwardRef, type Ref } from "react";

interface InputProps {
  label: string;
}

const LabeledInput = forwardRef<HTMLInputElement, InputProps>(
  function LabeledInput(props, ref) {
    return (
      <label>
        {props.label}
        <input ref={ref} />
      </label>
    );
  }
);

// Usage
const ref = useRef<HTMLInputElement>(null);
<LabeledInput ref={ref} label="Name" />;
```

### React 19: ref as Regular Prop

```typescript
import { type Ref } from "react";

interface InputProps {
  label: string;
  ref?: Ref<HTMLInputElement>;
}

function LabeledInput({ label, ref }: InputProps): JSX.Element {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}

// Usage — identical
const ref = useRef<HTMLInputElement>(null);
<LabeledInput ref={ref} label="Name" />;
```

In React 19, `ref` is destructured from props like any other prop. No wrapper needed.

---

## useImperativeHandle

Exposes a limited, custom API to the parent instead of the raw DOM node.

### React 19 Pattern

```typescript
import { useRef, useImperativeHandle, type Ref } from "react";

interface TextInputHandle {
  focus: () => void;
  scrollIntoView: () => void;
}

interface TextInputProps {
  placeholder?: string;
  ref?: Ref<TextInputHandle>;
}

function TextInput({ placeholder, ref }: TextInputProps): JSX.Element {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current?.focus();
    },
    scrollIntoView() {
      inputRef.current?.scrollIntoView({ behavior: "smooth" });
    },
  }), []);

  return <input ref={inputRef} placeholder={placeholder} />;
}

// Parent usage
function Parent(): JSX.Element {
  const textInputRef = useRef<TextInputHandle>(null);

  return (
    <>
      <TextInput ref={textInputRef} placeholder="Type here" />
      <button onClick={() => textInputRef.current?.focus()}>
        Focus Input
      </button>
    </>
  );
}
```

### React 18 Pattern

```typescript
import { forwardRef, useRef, useImperativeHandle } from "react";

const TextInput = forwardRef<TextInputHandle, TextInputProps>(
  function TextInput(props, ref) {
    const inputRef = useRef<HTMLInputElement>(null);
    useImperativeHandle(ref, () => ({
      focus() { inputRef.current?.focus(); },
    }), []);
    return <input ref={inputRef} placeholder={props.placeholder} />;
  }
);
```

ALWAYS define a TypeScript interface for the handle type. This provides compile-time safety for the parent component.

---

## Callback Refs

A function passed as `ref` receives the DOM node on mount and `null` on unmount.

### Basic Callback Ref

```typescript
function MeasuredBox(): JSX.Element {
  const [height, setHeight] = useState<number>(0);

  const measuredRef = (node: HTMLDivElement | null): void => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  };

  return <div ref={measuredRef}>Height: {height}px</div>;
}
```

### React 19: Ref Cleanup Function

In React 19, callback refs can return a cleanup function (like effects):

```typescript
function VideoPlayer(): JSX.Element {
  const onRefChange = (node: HTMLVideoElement | null): (() => void) | void => {
    if (node !== null) {
      node.play();
      return () => {
        node.pause(); // Cleanup when ref detaches
      };
    }
  };

  return <video ref={onRefChange} src="/video.mp4" />;
}
```

**React 18**: Callback ref receives `null` on unmount. No return value.
**React 19**: Callback ref MAY return a cleanup function. React calls it when the node detaches. Returning a cleanup means the ref does NOT receive `null` on unmount.

---

## Ref Read/Write Rules

| Context | Read `.current` | Write `.current` |
|---------|----------------|-----------------|
| Event handlers | ALLOWED | ALLOWED |
| Effects (`useEffect`, `useLayoutEffect`) | ALLOWED | ALLOWED |
| During render | FORBIDDEN | FORBIDDEN |
| Lazy initialization in render | ALLOWED (`if (ref.current === null)`) | ALLOWED (one-time init) |

### Lazy Initialization Pattern

```typescript
function getPlayer(): VideoPlayer {
  // Expensive to create — initialize once
  return new VideoPlayer();
}

function VideoComponent(): JSX.Element {
  const playerRef = useRef<VideoPlayer | null>(null);

  // Safe: one-time initialization check during render
  if (playerRef.current === null) {
    playerRef.current = getPlayer();
  }

  return <div>{/* use playerRef.current */}</div>;
}
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Working ref patterns with full TypeScript types
- [references/anti-patterns.md](references/anti-patterns.md) -- Common ref mistakes and corrections

### Official Sources

- https://react.dev/reference/react/useRef
- https://react.dev/reference/react/useImperativeHandle
- https://react.dev/reference/react/forwardRef
- https://react.dev/learn/referencing-values-with-refs
- https://react.dev/learn/manipulating-the-dom-with-refs
