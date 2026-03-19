# react-syntax-refs — Examples

> Extended ref patterns with full TypeScript types. All examples verified against React 18.x and 19.x official documentation.

---

## Scrolling to a List Item with Callback Refs

Using a `Map` to track multiple DOM nodes without knowing the count at compile time:

```typescript
import { useRef } from "react";

interface CatItem {
  id: string;
  imageUrl: string;
}

function CatGallery({ cats }: { cats: CatItem[] }): JSX.Element {
  const itemsRef = useRef<Map<string, HTMLLIElement>>(new Map());

  function scrollToId(itemId: string): void {
    const node = itemsRef.current.get(itemId);
    node?.scrollIntoView({ behavior: "smooth", block: "nearest" });
  }

  return (
    <ul>
      {cats.map((cat) => (
        <li
          key={cat.id}
          ref={(node: HTMLLIElement | null) => {
            if (node !== null) {
              itemsRef.current.set(cat.id, node);
            } else {
              itemsRef.current.delete(cat.id);
            }
          }}
        >
          <img src={cat.imageUrl} alt={`Cat ${cat.id}`} />
        </li>
      ))}
    </ul>
  );
}
```

**When to use**: ALWAYS use a `Map`-based callback ref when you need refs to a dynamic list of elements. A single `useRef` can only hold one value.

---

## Previous Value Tracking

```typescript
import { useRef, useEffect } from "react";

function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage
function Counter({ count }: { count: number }): JSX.Element {
  const prevCount = usePrevious(count);
  return (
    <p>
      Now: {count}, Before: {prevCount ?? "N/A"}
    </p>
  );
}
```

---

## Stable Callback with useRef

Storing the latest version of a callback without causing effect re-runs:

```typescript
import { useRef, useEffect, useCallback } from "react";

function useLatestCallback<T extends (...args: any[]) => any>(
  callback: T
): T {
  const ref = useRef<T>(callback);

  useEffect(() => {
    ref.current = callback;
  }, [callback]);

  // Return a stable wrapper that always calls the latest version
  return useCallback(
    ((...args: Parameters<T>) => ref.current(...args)) as T,
    []
  );
}
```

---

## Composing Multiple Refs

When a single element needs multiple refs (e.g., a library ref and a local ref):

```typescript
import { useRef, useCallback, type Ref, type RefCallback } from "react";

function useMergedRefs<T>(
  ...refs: Array<Ref<T> | undefined>
): RefCallback<T> {
  return useCallback((node: T | null) => {
    refs.forEach((ref) => {
      if (ref === null || ref === undefined) return;
      if (typeof ref === "function") {
        ref(node);
      } else {
        (ref as React.MutableRefObject<T | null>).current = node;
      }
    });
  }, refs);
}

// Usage
function InputWithLibrary({
  ref,
}: {
  ref?: Ref<HTMLInputElement>;
}): JSX.Element {
  const localRef = useRef<HTMLInputElement>(null);
  const mergedRef = useMergedRefs(ref, localRef);

  return <input ref={mergedRef} />;
}
```

---

## useImperativeHandle with Complex API

Exposing a rich imperative API for a modal component:

```typescript
import { useRef, useImperativeHandle, useState, type Ref } from "react";

interface ModalHandle {
  open: () => void;
  close: () => void;
  toggle: () => void;
  isOpen: () => boolean;
}

interface ModalProps {
  title: string;
  children: React.ReactNode;
  ref?: Ref<ModalHandle>;
}

function Modal({ title, children, ref }: ModalProps): JSX.Element | null {
  const [visible, setVisible] = useState<boolean>(false);

  useImperativeHandle(ref, () => ({
    open() {
      setVisible(true);
    },
    close() {
      setVisible(false);
    },
    toggle() {
      setVisible((prev) => !prev);
    },
    isOpen() {
      return visible;
    },
  }), [visible]);

  if (!visible) return null;

  return (
    <div role="dialog" aria-label={title}>
      <h2>{title}</h2>
      {children}
      <button onClick={() => setVisible(false)}>Close</button>
    </div>
  );
}

// Parent usage
function App(): JSX.Element {
  const modalRef = useRef<ModalHandle>(null);

  return (
    <>
      <button onClick={() => modalRef.current?.open()}>Show Modal</button>
      <Modal ref={modalRef} title="Confirm">
        <p>Are you sure?</p>
      </Modal>
    </>
  );
}
```

---

## Conditional Ref Assignment

Using callback refs to conditionally track elements:

```typescript
import { useState, useCallback } from "react";

function AutofocusOnExpand(): JSX.Element {
  const [expanded, setExpanded] = useState<boolean>(false);

  const inputCallbackRef = useCallback((node: HTMLInputElement | null) => {
    if (node !== null) {
      // Focus immediately when the input mounts
      node.focus();
    }
  }, []);

  return (
    <div>
      <button onClick={() => setExpanded((prev) => !prev)}>
        {expanded ? "Collapse" : "Expand"}
      </button>
      {expanded && <input ref={inputCallbackRef} placeholder="Auto-focused" />}
    </div>
  );
}
```

**When to use**: ALWAYS use a callback ref when you need to perform an action (focus, measure, animate) at the exact moment a DOM node attaches.

---

## React 19 Ref Cleanup for Intersection Observer

```typescript
function LazyImage({ src, alt }: { src: string; alt: string }): JSX.Element {
  const [isVisible, setIsVisible] = useState<boolean>(false);

  // React 19: return cleanup from callback ref
  const observeRef = (node: HTMLDivElement | null): (() => void) | void => {
    if (node === null) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    observer.observe(node);

    return () => {
      observer.disconnect(); // Cleanup on detach
    };
  };

  return (
    <div ref={observeRef}>
      {isVisible ? <img src={src} alt={alt} /> : <div>Loading...</div>}
    </div>
  );
}
```

---

## DOM Measurement with useLayoutEffect

When you need accurate measurements before the browser paints:

```typescript
import { useRef, useState, useLayoutEffect } from "react";

interface Dimensions {
  width: number;
  height: number;
}

function MeasuredContainer({
  children,
}: {
  children: React.ReactNode;
}): JSX.Element {
  const containerRef = useRef<HTMLDivElement>(null);
  const [dimensions, setDimensions] = useState<Dimensions>({
    width: 0,
    height: 0,
  });

  useLayoutEffect(() => {
    if (containerRef.current !== null) {
      const { width, height } =
        containerRef.current.getBoundingClientRect();
      setDimensions({ width, height });
    }
  }, [children]);

  return (
    <div ref={containerRef}>
      <div>
        Size: {dimensions.width}x{dimensions.height}
      </div>
      {children}
    </div>
  );
}
```

ALWAYS use `useLayoutEffect` (not `useEffect`) when measurements must be accurate before paint. `useEffect` runs after paint and causes visible flicker.

---

## Forwarding Refs Through HOCs (React 18)

```typescript
import { forwardRef, type ComponentType, type Ref } from "react";

interface WithLoggerProps {
  ref?: Ref<HTMLElement>;
}

function withLogger<P extends object>(
  WrappedComponent: ComponentType<P>
): ComponentType<P & WithLoggerProps> {
  const WithLogger = forwardRef<HTMLElement, P>(
    function WithLogger(props, ref) {
      console.log("Rendered:", WrappedComponent.displayName);
      return <WrappedComponent {...props} ref={ref} />;
    }
  );

  WithLogger.displayName = `WithLogger(${
    WrappedComponent.displayName ?? WrappedComponent.name
  })`;

  return WithLogger;
}
```

In React 19 this HOC pattern is simplified because `ref` flows through as a normal prop without `forwardRef`.
