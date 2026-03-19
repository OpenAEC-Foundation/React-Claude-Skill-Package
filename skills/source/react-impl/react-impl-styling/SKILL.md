---
name: react-impl-styling
description: >
  Use when styling React components, choosing a CSS approach, implementing
  responsive design, or managing CSS architecture. Prevents the common mistake
  of global style leakage or choosing an incompatible CSS-in-JS library for
  Server Components. Covers CSS Modules, Tailwind CSS, CSS-in-JS, inline styles,
  clsx/cn, conditional classes, CSS custom properties.
  Keywords: CSS Modules, Tailwind, CSS-in-JS, clsx, className, scoped styles.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-styling

## Quick Reference

### Styling Approach Decision Tree

```
Need to style a React component?
├── Is it a dynamic value computed at runtime (e.g., position, color from props)?
│   └── YES → Use inline styles with React.CSSProperties
├── Is the project using Tailwind CSS?
│   └── YES → Use Tailwind utility classes with cn() helper
├── Do you need scoped, file-level styles?
│   └── YES → Use CSS Modules (.module.css)
├── Do you need a component library with runtime theming?
│   └── YES → Consider CSS-in-JS (styled-components / Emotion)
└── DEFAULT → Use CSS Modules (safest, zero-runtime, best performance)
```

### Approach Comparison

| Approach | Runtime Cost | Scoping | TypeScript Support | Recommendation |
|----------|-------------|---------|-------------------|----------------|
| CSS Modules | None | Automatic | Via declarations | **Primary** |
| Tailwind CSS | None | Via utilities | Via cn() typing | **Strong alternative** |
| Inline styles | Minimal | Inline | Native | Dynamic values only |
| CSS-in-JS | Moderate | Automatic | Native | Only when needed |
| Global CSS | None | None (global) | None | Variables + resets only |

### Critical Warnings

**NEVER** use inline styles for pseudo-classes (`:hover`, `:focus`), media queries, or animations -- inline styles cannot express these. ALWAYS use CSS Modules or Tailwind for interactive/responsive styles.

**NEVER** use string interpolation to build className strings -- ALWAYS use `clsx()` or `cn()` for conditional class composition to avoid whitespace bugs and improve readability.

**NEVER** import `.css` files in component files without the `.module.css` suffix when you need scoping -- plain `.css` imports are global and WILL cause style collisions across components.

**NEVER** use CSS-in-JS (styled-components, Emotion) in React Server Components -- these libraries require a client-side runtime. ALWAYS use CSS Modules or Tailwind for RSC.

**ALWAYS** define a TypeScript module declaration for `.module.css` files to prevent import errors and enable autocomplete.

---

## CSS Modules (Primary Recommendation)

CSS Modules provide automatic scoping with zero runtime cost. The bundler (Vite, webpack) transforms class names to unique hashes at build time.

### Setup

Create a TypeScript declaration so imports are typed:

```typescript
// src/types/css.d.ts
declare module "*.module.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}
```

### Basic Usage

```tsx
// Button.module.css
.button {
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-weight: 600;
  cursor: pointer;
}

.primary {
  background-color: var(--color-primary);
  color: white;
}

.secondary {
  background-color: transparent;
  border: 1px solid var(--color-primary);
  color: var(--color-primary);
}
```

```tsx
// Button.tsx
import styles from "./Button.module.css";
import { clsx } from "clsx";

interface ButtonProps {
  variant?: "primary" | "secondary";
  children: React.ReactNode;
}

export function Button({ variant = "primary", children }: ButtonProps) {
  return (
    <button className={clsx(styles.button, styles[variant])}>
      {children}
    </button>
  );
}
```

### Composition with `composes`

```css
/* Card.module.css */
.base {
  border-radius: 0.5rem;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12);
}

.elevated {
  composes: base;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.interactive {
  composes: base;
  cursor: pointer;
  transition: box-shadow 0.2s ease;
}
```

---

## Tailwind CSS Integration

### cn() Utility (clsx + tailwind-merge)

ALWAYS create a `cn()` utility that combines `clsx` for conditional classes with `tailwind-merge` to resolve Tailwind class conflicts:

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

### Component Pattern

```tsx
// Button.tsx
import { cn } from "@/lib/utils";

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
  size?: "sm" | "md" | "lg";
}

const variantStyles = {
  primary: "bg-blue-600 text-white hover:bg-blue-700",
  secondary: "bg-transparent border border-blue-600 text-blue-600 hover:bg-blue-50",
} as const;

const sizeStyles = {
  sm: "px-3 py-1.5 text-sm",
  md: "px-4 py-2 text-base",
  lg: "px-6 py-3 text-lg",
} as const;

export function Button({
  variant = "primary",
  size = "md",
  className,
  children,
  ...props
}: ButtonProps) {
  return (
    <button
      className={cn(
        "rounded-md font-semibold transition-colors",
        variantStyles[variant],
        sizeStyles[size],
        className
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

### Responsive Design with Tailwind

Tailwind uses mobile-first breakpoints. ALWAYS design mobile-first and add breakpoint prefixes for larger screens:

```tsx
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  {items.map((item) => (
    <Card key={item.id} className="p-4 sm:p-6" />
  ))}
</div>
```

| Prefix | Min-width | Typical Device |
|--------|-----------|---------------|
| (none) | 0px | Mobile |
| `sm:` | 640px | Large phone / small tablet |
| `md:` | 768px | Tablet |
| `lg:` | 1024px | Laptop |
| `xl:` | 1280px | Desktop |
| `2xl:` | 1536px | Large desktop |

### Dark Mode with Tailwind

```tsx
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
  <h1 className="text-2xl font-bold text-gray-800 dark:text-gray-200">
    Dashboard
  </h1>
</div>
```

ALWAYS pair light and dark variants together on the same element for maintainability.

---

## Inline Styles (Dynamic Values Only)

ALWAYS type inline styles with `React.CSSProperties`. Use inline styles ONLY for values computed at runtime from props or state:

```tsx
interface ProgressBarProps {
  value: number; // 0-100
  color?: string;
}

export function ProgressBar({ value, color = "#3b82f6" }: ProgressBarProps) {
  const barStyle: React.CSSProperties = {
    width: `${Math.min(100, Math.max(0, value))}%`,
    backgroundColor: color,
    height: "0.5rem",
    borderRadius: "0.25rem",
    transition: "width 0.3s ease",
  };

  return (
    <div style={{ backgroundColor: "#e5e7eb", borderRadius: "0.25rem" }}>
      <div style={barStyle} role="progressbar" aria-valuenow={value} />
    </div>
  );
}
```

**NEVER** use inline styles for static values -- move them to CSS Modules or Tailwind classes instead. Inline styles bypass the cascade, cannot be overridden by consumers, and increase bundle size.

---

## className Patterns with clsx

The `clsx` library builds className strings from conditional inputs. ALWAYS use it instead of manual string concatenation:

```tsx
import { clsx } from "clsx";

interface AlertProps {
  severity: "info" | "warning" | "error";
  dismissible?: boolean;
  children: React.ReactNode;
}

export function Alert({ severity, dismissible = false, children }: AlertProps) {
  return (
    <div
      className={clsx(
        "rounded-md p-4 text-sm",
        {
          "bg-blue-50 text-blue-700": severity === "info",
          "bg-yellow-50 text-yellow-700": severity === "warning",
          "bg-red-50 text-red-700": severity === "error",
        },
        dismissible && "pr-10"
      )}
      role="alert"
    >
      {children}
    </div>
  );
}
```

---

## Global Styles and CSS Custom Properties

### Global CSS Setup

ALWAYS define global design tokens as CSS custom properties in `:root`. This is the ONLY appropriate use for global CSS files:

```css
/* src/index.css */
:root {
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-secondary: #64748b;
  --color-background: #ffffff;
  --color-surface: #f8fafc;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-border: #e2e8f0;
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.07);
}

/* Dark mode override */
@media (prefers-color-scheme: dark) {
  :root {
    --color-background: #0f172a;
    --color-surface: #1e293b;
    --color-text: #f1f5f9;
    --color-text-muted: #94a3b8;
    --color-border: #334155;
  }
}

/* Reset */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
}
```

### Using Custom Properties in CSS Modules

```css
/* Card.module.css */
.card {
  background-color: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: 1.5rem;
  box-shadow: var(--shadow-sm);
  color: var(--color-text);
}
```

---

## Responsive Design with CSS Modules

Use standard media queries inside CSS Modules. ALWAYS use mobile-first (`min-width`) breakpoints:

```css
/* Layout.module.css */
.grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
}

@media (min-width: 640px) { .grid { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 1024px) { .grid { grid-template-columns: repeat(3, 1fr); } }
```

---

## Dark Mode with CSS Custom Properties

For non-Tailwind projects, use CSS custom properties with a `data-theme` attribute and `prefers-color-scheme` fallback. Define dark overrides in `:root[data-theme="dark"]` (see Global Styles section above for the pattern). Toggle with:

```tsx
import { useEffect, useState } from "react";

type Theme = "light" | "dark" | "system";

export function useTheme() {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem("theme") as Theme) ?? "system"
  );

  useEffect(() => {
    const root = document.documentElement;
    if (theme === "system") root.removeAttribute("data-theme");
    else root.setAttribute("data-theme", theme);
    localStorage.setItem("theme", theme);
  }, [theme]);

  return { theme, setTheme };
}
```

---

## React 19: Stylesheet Precedence

React 19 introduces built-in support for `<link>` stylesheet ordering. Use the `precedence` prop to control CSS load order without manual management:

```tsx
// React 19 only
function ProductPage() {
  return (
    <>
      <link rel="stylesheet" href="/base.css" precedence="default" />
      <link rel="stylesheet" href="/product.css" precedence="high" />
      <div className="product-layout">
        <ProductDetails />
      </div>
    </>
  );
}

function ProductDetails() {
  // This stylesheet is deduplicated -- React loads it only once
  return (
    <>
      <link rel="stylesheet" href="/product.css" precedence="high" />
      <div className="product-details">{/* ... */}</div>
    </>
  );
}
```

React 19 deduplicates stylesheet links and orders them by precedence. This eliminates the need for CSS-in-JS runtime ordering in many cases.

**React 18**: No `precedence` prop support. Use CSS Modules or manual `<link>` ordering in `index.html`.

---

## CSS-in-JS Overview

CSS-in-JS libraries (styled-components, Emotion) co-locate styles with components and support dynamic theming. Use them ONLY when the project requires runtime theme switching or an existing codebase depends on them.

```tsx
// styled-components pattern (brief reference)
import styled from "styled-components";

const StyledButton = styled.button<{ $variant: "primary" | "secondary" }>`
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-weight: 600;
  background-color: ${(props) =>
    props.$variant === "primary" ? "var(--color-primary)" : "transparent"};
  color: ${(props) =>
    props.$variant === "primary" ? "white" : "var(--color-primary)"};
`;
```

**NEVER** choose CSS-in-JS for new projects without a specific runtime theming requirement. CSS Modules and Tailwind cover the vast majority of use cases with zero runtime cost.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete styling approach examples with TypeScript
- [references/anti-patterns.md](references/anti-patterns.md) -- Common styling mistakes and how to avoid them

### Official Sources

- https://react.dev/learn#adding-styles
- https://react.dev/reference/react-dom/components/link (React 19 precedence)
- https://tailwindcss.com/docs
- https://github.com/lukeed/clsx
- https://github.com/dcastil/tailwind-merge
