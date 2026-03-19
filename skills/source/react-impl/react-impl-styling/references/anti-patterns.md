# react-impl-styling — Anti-Patterns

## AP-01: String Concatenation for className

**NEVER** build className strings with template literals or concatenation.

```tsx
// BAD -- whitespace bugs, hard to read, no type safety
function Badge({ active, size }: { active: boolean; size: string }) {
  return (
    <span className={`badge ${active ? "badge-active" : ""} badge-${size}`}>
      {/* Extra space when active is false: "badge  badge-sm" */}
    </span>
  );
}

// GOOD -- use clsx for clean conditional composition
import { clsx } from "clsx";

function Badge({ active, size }: { active: boolean; size: "sm" | "md" | "lg" }) {
  return (
    <span
      className={clsx("badge", active && "badge-active", `badge-${size}`)}
    />
  );
}
```

**Why**: String concatenation produces extra whitespace when conditions are false, is harder to read with multiple conditions, and provides no static analysis benefits.

---

## AP-02: Inline Styles for Static Values

**NEVER** use inline styles for values that do not change at runtime.

```tsx
// BAD -- static styles in inline object, recreated every render
function Sidebar() {
  return (
    <aside
      style={{
        width: "16rem",
        backgroundColor: "#f8fafc",
        borderRight: "1px solid #e2e8f0",
        padding: "1.5rem",
        minHeight: "100vh",
      }}
    >
      {/* ... */}
    </aside>
  );
}

// GOOD -- static styles in CSS Module
// Sidebar.module.css:
// .sidebar { width: 16rem; background-color: var(--color-surface); ... }

import styles from "./Sidebar.module.css";

function Sidebar() {
  return <aside className={styles.sidebar}>{/* ... */}</aside>;
}
```

**Why**: Inline style objects are recreated every render (new object reference), cannot use pseudo-classes or media queries, cannot be overridden by consumers, and bloat the HTML output.

---

## AP-03: Global CSS Without Scoping

**NEVER** import plain `.css` files in component files for component-specific styles.

```tsx
// BAD -- .button class is global, will collide with other .button classes
import "./Button.css"; // NOT .module.css

function Button() {
  return <button className="button">Click</button>;
}

// GOOD -- CSS Modules scope automatically
import styles from "./Button.module.css";

function Button() {
  return <button className={styles.button}>Click</button>;
}
```

**Why**: Plain CSS imports are global. Two components defining `.card` or `.button` will overwrite each other's styles unpredictably based on import order.

---

## AP-04: CSS-in-JS in Server Components

**NEVER** use styled-components, Emotion, or other CSS-in-JS runtime libraries in React Server Components.

```tsx
// BAD -- crashes in RSC because styled-components requires browser APIs
"use server"; // or no directive in app/ directory (RSC by default)
import styled from "styled-components";

const Container = styled.div`
  max-width: 1200px;
  margin: 0 auto;
`;

// GOOD -- use CSS Modules in Server Components
import styles from "./Layout.module.css";

export function Layout({ children }: { children: React.ReactNode }) {
  return <div className={styles.container}>{children}</div>;
}
```

**Why**: CSS-in-JS libraries inject styles at runtime using browser APIs (`document.createElement`, `insertRule`). These APIs do not exist on the server. Server Components NEVER run on the client.

---

## AP-05: Tailwind Classes Without tailwind-merge

**NEVER** use `clsx` alone when merging Tailwind classes from external props -- conflicting utilities will both apply.

```tsx
// BAD -- if className="p-8", both p-4 AND p-8 apply (unpredictable)
import { clsx } from "clsx";

function Card({ className }: { className?: string }) {
  return <div className={clsx("rounded-lg p-4 bg-white", className)} />;
}

// GOOD -- tailwind-merge resolves conflicts (p-8 wins over p-4)
import { cn } from "@/lib/utils"; // clsx + twMerge

function Card({ className }: { className?: string }) {
  return <div className={cn("rounded-lg p-4 bg-white", className)} />;
}
```

**Why**: Tailwind utilities are atomic CSS classes. When two conflicting utilities exist (`p-4` and `p-8`), the winner depends on CSS source order, not DOM order. `tailwind-merge` intelligently resolves conflicts by keeping only the last value for each CSS property.

---

## AP-06: Hardcoded Colors Instead of Design Tokens

**NEVER** hardcode color values directly in component styles.

```css
/* BAD -- hardcoded hex values scattered across files */
.header {
  background-color: #1e293b;
  color: #f1f5f9;
  border-bottom: 1px solid #334155;
}

/* GOOD -- design tokens via CSS custom properties */
.header {
  background-color: var(--color-surface);
  color: var(--color-text);
  border-bottom: 1px solid var(--color-border);
}
```

**Why**: Hardcoded values make theming impossible, dark mode support painful, and design changes require find-and-replace across the entire codebase. CSS custom properties provide a single source of truth.

---

## AP-07: Using style Prop for Pseudo-Classes

**NEVER** try to implement hover, focus, or active states with inline styles and event handlers.

```tsx
// BAD -- recreating CSS with JavaScript, verbose, buggy, inaccessible
function BadButton() {
  const [hovered, setHovered] = useState(false);

  return (
    <button
      style={{
        backgroundColor: hovered ? "#2563eb" : "#3b82f6",
        color: "white",
        padding: "0.5rem 1rem",
      }}
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
    >
      Click me
    </button>
  );
}

// GOOD -- CSS handles pseudo-classes natively
// Button.module.css:
// .button { background-color: var(--color-primary); }
// .button:hover { background-color: var(--color-primary-hover); }
// .button:focus-visible { outline: 2px solid var(--color-primary); }

import styles from "./Button.module.css";

function GoodButton() {
  return <button className={styles.button}>Click me</button>;
}
```

**Why**: JavaScript hover handlers miss keyboard focus states, do not handle touch correctly, cause unnecessary re-renders, and cannot handle `:focus-visible` or `:active` properly. CSS pseudo-classes are declarative, accessible, and performant.

---

## AP-08: Mixing Styling Approaches Inconsistently

**NEVER** mix CSS Modules, Tailwind, and inline styles randomly within the same component.

```tsx
// BAD -- three different styling approaches in one component
import styles from "./Card.module.css";

function Card({ highlight }: { highlight: boolean }) {
  return (
    <div
      className={`${styles.card} flex flex-col`}
      style={{ borderColor: highlight ? "red" : "gray" }}
    >
      <h2 className="text-lg font-bold">{/* Tailwind */}</h2>
      <p className={styles.body}>{/* CSS Module */}</p>
    </div>
  );
}

// GOOD -- pick ONE primary approach and use it consistently
import { cn } from "@/lib/utils";

function Card({ highlight }: { highlight: boolean }) {
  return (
    <div
      className={cn(
        "flex flex-col rounded-lg border p-4",
        highlight ? "border-red-500" : "border-gray-300"
      )}
    >
      <h2 className="text-lg font-bold">{/* ... */}</h2>
      <p className="text-gray-600">{/* ... */}</p>
    </div>
  );
}
```

**Why**: Mixing approaches makes styles unpredictable (specificity wars), harder to maintain, and confusing for other developers. ALWAYS pick one primary approach per project.

---

## AP-09: Not Forwarding className to Root Element

**NEVER** create reusable components that ignore the consumer's `className` prop.

```tsx
// BAD -- consumers cannot customize the component
function Card({ children }: { children: React.ReactNode }) {
  return <div className="rounded-lg border p-4">{children}</div>;
}

// Consumer: <Card className="mt-8" /> -- mt-8 is silently ignored!

// GOOD -- ALWAYS merge consumer className with internal classes
import { cn } from "@/lib/utils";

interface CardProps {
  children: React.ReactNode;
  className?: string;
}

function Card({ children, className }: CardProps) {
  return (
    <div className={cn("rounded-lg border p-4", className)}>
      {children}
    </div>
  );
}
```

**Why**: Reusable components MUST accept and forward `className` so consumers can control spacing, positioning, and overrides. Without this, every layout adjustment requires wrapper divs.

---

## AP-10: Using !important to Fix Specificity

**NEVER** use `!important` to fix styling conflicts between components.

```css
/* BAD -- specificity arms race */
.button {
  padding: 0.5rem 1rem !important;
  background-color: blue !important;
}

/* GOOD -- fix the root cause: use CSS Modules for scoping */
/* Or use CSS layers for controlled specificity */
@layer components {
  .button {
    padding: 0.5rem 1rem;
    background-color: blue;
  }
}
```

**Why**: `!important` creates a specificity escalation that makes styles progressively harder to override. It indicates a scoping problem that should be solved with CSS Modules, CSS layers, or restructuring selectors.
