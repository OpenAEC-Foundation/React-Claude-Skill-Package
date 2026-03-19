# react-impl-styling — Examples

## CSS Modules: Complete Component System

### Module Declaration

```typescript
// src/types/css.d.ts
declare module "*.module.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.module.scss" {
  const classes: { readonly [key: string]: string };
  export default classes;
}
```

### Card Component with CSS Modules

```css
/* Card.module.css */
.card {
  background-color: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.header {
  padding: 1rem 1.5rem;
  border-bottom: 1px solid var(--color-border);
  font-weight: 600;
  font-size: 1.125rem;
  color: var(--color-text);
}

.body {
  padding: 1.5rem;
  color: var(--color-text);
}

.footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid var(--color-border);
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
}

.clickable {
  cursor: pointer;
  transition: box-shadow 0.2s ease, transform 0.2s ease;
}

.clickable:hover {
  box-shadow: var(--shadow-md);
  transform: translateY(-1px);
}

.clickable:active {
  transform: translateY(0);
}
```

```tsx
// Card.tsx
import styles from "./Card.module.css";
import { clsx } from "clsx";

interface CardProps {
  children: React.ReactNode;
  onClick?: () => void;
  className?: string;
}

interface CardHeaderProps {
  children: React.ReactNode;
}

interface CardBodyProps {
  children: React.ReactNode;
}

interface CardFooterProps {
  children: React.ReactNode;
}

export function Card({ children, onClick, className }: CardProps) {
  const isClickable = typeof onClick === "function";
  const Tag = isClickable ? "button" : "div";

  return (
    <Tag
      className={clsx(styles.card, isClickable && styles.clickable, className)}
      onClick={onClick}
      type={isClickable ? "button" : undefined}
    >
      {children}
    </Tag>
  );
}

export function CardHeader({ children }: CardHeaderProps) {
  return <div className={styles.header}>{children}</div>;
}

export function CardBody({ children }: CardBodyProps) {
  return <div className={styles.body}>{children}</div>;
}

export function CardFooter({ children }: CardFooterProps) {
  return <div className={styles.footer}>{children}</div>;
}
```

### Usage

```tsx
<Card onClick={() => navigate(`/project/${project.id}`)}>
  <CardHeader>Project Settings</CardHeader>
  <CardBody>
    <p>Configure project-level settings and permissions.</p>
  </CardBody>
  <CardFooter>
    <Button variant="secondary">Cancel</Button>
    <Button variant="primary">Save</Button>
  </CardFooter>
</Card>
```

---

## Tailwind CSS: Complete Component System

### cn() Utility Setup

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

### Badge Component

```tsx
// Badge.tsx
import { cn } from "@/lib/utils";

type BadgeVariant = "default" | "success" | "warning" | "error" | "info";

interface BadgeProps {
  variant?: BadgeVariant;
  children: React.ReactNode;
  className?: string;
}

const variantStyles: Record<BadgeVariant, string> = {
  default: "bg-gray-100 text-gray-800 dark:bg-gray-800 dark:text-gray-200",
  success: "bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200",
  warning: "bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200",
  error: "bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200",
  info: "bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200",
};

export function Badge({ variant = "default", children, className }: BadgeProps) {
  return (
    <span
      className={cn(
        "inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium",
        variantStyles[variant],
        className
      )}
    >
      {children}
    </span>
  );
}
```

### Responsive Navigation

```tsx
// Navigation.tsx
import { useState } from "react";
import { cn } from "@/lib/utils";

interface NavItem {
  label: string;
  href: string;
}

interface NavigationProps {
  items: NavItem[];
}

export function Navigation({ items }: NavigationProps) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <nav className="relative bg-white shadow dark:bg-gray-900">
      <div className="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
        <div className="flex h-16 items-center justify-between">
          <div className="flex-shrink-0">
            <span className="text-xl font-bold text-gray-900 dark:text-white">
              Logo
            </span>
          </div>

          {/* Desktop menu */}
          <div className="hidden sm:flex sm:items-center sm:gap-4">
            {items.map((item) => (
              <a
                key={item.href}
                href={item.href}
                className="rounded-md px-3 py-2 text-sm font-medium text-gray-700 hover:bg-gray-100 hover:text-gray-900 dark:text-gray-300 dark:hover:bg-gray-800 dark:hover:text-white"
              >
                {item.label}
              </a>
            ))}
          </div>

          {/* Mobile menu button */}
          <button
            type="button"
            className="inline-flex items-center justify-center rounded-md p-2 text-gray-700 hover:bg-gray-100 sm:hidden dark:text-gray-300 dark:hover:bg-gray-800"
            onClick={() => setIsOpen((prev) => !prev)}
            aria-expanded={isOpen}
          >
            <span className="sr-only">Open main menu</span>
            {isOpen ? "Close" : "Menu"}
          </button>
        </div>
      </div>

      {/* Mobile menu */}
      <div className={cn("sm:hidden", isOpen ? "block" : "hidden")}>
        <div className="space-y-1 px-2 pb-3 pt-2">
          {items.map((item) => (
            <a
              key={item.href}
              href={item.href}
              className="block rounded-md px-3 py-2 text-base font-medium text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-800"
            >
              {item.label}
            </a>
          ))}
        </div>
      </div>
    </nav>
  );
}
```

---

## Inline Styles: Dynamic Value Patterns

### Animated Progress Ring

```tsx
interface ProgressRingProps {
  value: number; // 0-100
  size?: number;
  strokeWidth?: number;
  color?: string;
}

export function ProgressRing({
  value,
  size = 80,
  strokeWidth = 6,
  color = "#3b82f6",
}: ProgressRingProps) {
  const radius = (size - strokeWidth) / 2;
  const circumference = radius * 2 * Math.PI;
  const offset = circumference - (value / 100) * circumference;

  return (
    <svg width={size} height={size} role="progressbar" aria-valuenow={value}>
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        fill="none"
        stroke="#e5e7eb"
        strokeWidth={strokeWidth}
      />
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        fill="none"
        stroke={color}
        strokeWidth={strokeWidth}
        strokeDasharray={circumference}
        strokeDashoffset={offset}
        strokeLinecap="round"
        style={{
          transition: "stroke-dashoffset 0.3s ease",
          transform: "rotate(-90deg)",
          transformOrigin: "50% 50%",
        }}
      />
    </svg>
  );
}
```

### Grid Layout from Data

```tsx
interface DashboardGridProps {
  columns: number;
  gap?: number;
  children: React.ReactNode;
}

export function DashboardGrid({ columns, gap = 16, children }: DashboardGridProps) {
  const gridStyle: React.CSSProperties = {
    display: "grid",
    gridTemplateColumns: `repeat(${columns}, 1fr)`,
    gap: `${gap}px`,
  };

  return <div style={gridStyle}>{children}</div>;
}
```

---

## CSS Custom Properties: Theming System

### Complete Theme Setup

```css
/* src/styles/tokens.css */
:root {
  /* Colors */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-primary-light: #dbeafe;
  --color-secondary: #8b5cf6;
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-error: #ef4444;

  /* Surfaces */
  --color-background: #ffffff;
  --color-surface: #f8fafc;
  --color-surface-elevated: #ffffff;

  /* Text */
  --color-text: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #94a3b8;
  --color-text-inverse: #ffffff;

  /* Borders */
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;

  /* Spacing scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Radii */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);

  /* Typography */
  --font-sans: system-ui, -apple-system, sans-serif;
  --font-mono: "JetBrains Mono", "Fira Code", monospace;
}

:root[data-theme="dark"] {
  --color-background: #0f172a;
  --color-surface: #1e293b;
  --color-surface-elevated: #334155;
  --color-text: #f1f5f9;
  --color-text-secondary: #cbd5e1;
  --color-text-muted: #64748b;
  --color-border: #334155;
  --color-border-strong: #475569;
  --color-primary-light: #1e3a5f;
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.3);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.4);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.5);
}
```

### Using Tokens in CSS Modules

```css
/* Input.module.css */
.input {
  width: 100%;
  padding: var(--space-2) var(--space-3);
  font-family: var(--font-sans);
  font-size: 0.875rem;
  line-height: 1.25rem;
  color: var(--color-text);
  background-color: var(--color-background);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  outline: none;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.input:focus {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-primary-light);
}

.input::placeholder {
  color: var(--color-text-muted);
}

.error {
  border-color: var(--color-error);
}

.error:focus {
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.2);
}
```

```tsx
// Input.tsx
import styles from "./Input.module.css";
import { clsx } from "clsx";
import { forwardRef } from "react";

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  error?: boolean;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ error = false, className, ...props }, ref) => {
    return (
      <input
        ref={ref}
        className={clsx(styles.input, error && styles.error, className)}
        aria-invalid={error}
        {...props}
      />
    );
  }
);

Input.displayName = "Input";
```

---

## React 19: Stylesheet Precedence Example

```tsx
// Layout.tsx (React 19)
function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <link rel="stylesheet" href="/styles/reset.css" precedence="reset" />
      <link rel="stylesheet" href="/styles/tokens.css" precedence="default" />
      <link rel="stylesheet" href="/styles/layout.css" precedence="default" />
      <main>{children}</main>
    </>
  );
}

// Feature.tsx (React 19) -- styles loaded on demand
function FeaturePage() {
  return (
    <>
      <link rel="stylesheet" href="/styles/feature.css" precedence="high" />
      <section className="feature-hero">{/* ... */}</section>
    </>
  );
}
```

React 19 handles deduplication automatically: if multiple components reference the same stylesheet href, it is loaded only once. The `precedence` value controls insertion order in `<head>`.
