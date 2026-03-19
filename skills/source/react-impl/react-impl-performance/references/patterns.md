# react-impl-performance — Patterns

## Pattern 1: React Compiler Setup (React 19+)

The React Compiler replaces manual `memo`, `useMemo`, and `useCallback` with automatic build-time memoization.

### Vite Setup

```bash
npm install -D babel-plugin-react-compiler@latest
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

### Vite with vite-plugin-babel

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import babel from 'vite-plugin-babel';

export default defineConfig({
  plugins: [
    react(),
    babel({
      babelConfig: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

### Babel (generic)

```js
// babel.config.js
module.exports = {
  plugins: [
    'babel-plugin-react-compiler', // MUST be first
    // ... other plugins
  ],
};
```

### Next.js

See https://nextjs.org/docs/app/api-reference/next-config-js/reactCompiler

### Webpack (community)

See https://github.com/SukkaW/react-compiler-webpack

### Configuration Options

```ts
const ReactCompilerConfig = {
  compilationMode: 'all',     // 'annotation' | 'infer' | 'all'
  target: '19',               // '17' | '18' | '19'
  panicThreshold: 'none',     // skip errors instead of failing build
};

// Pass as second element in plugin array:
plugins: [['babel-plugin-react-compiler', ReactCompilerConfig]];
```

| Mode | Behavior |
|------|----------|
| `'all'` | Compile every component and hook |
| `'infer'` | Compiler decides what to optimize |
| `'annotation'` | Only compile functions with `"use memo"` directive |

### ESLint Integration

```bash
npm install -D eslint-plugin-react-hooks@latest
```

ALWAYS install the ESLint plugin. It identifies Rules of React violations that prevent the compiler from optimizing components.

### Gradual Adoption with `"use memo"` and `"use no memo"`

```tsx
// Opt in a single component (when compilationMode is 'annotation')
function OptimizedComponent() {
  "use memo";
  return <div>Compiled</div>;
}

// Opt out a single component (any compilationMode)
function SkippedComponent() {
  "use no memo";
  return <div>Not compiled</div>;
}
```

Directives can be placed at module level (top of file, before imports) to apply to all functions in that file.

---

## Pattern 2: Stable Props for Memoized Children

When passing props to a `memo`-wrapped child, ALWAYS ensure referential stability.

### Strategy: Pass primitives instead of objects

```tsx
// BAD: New object every render
<Profile person={{ name, age }} />

// GOOD: Primitives are stable by value
<Profile name={name} age={age} />
```

### Strategy: useMemo for object props

```tsx
const person = useMemo(() => ({ name, age }), [name, age]);
<MemoizedProfile person={person} />
```

### Strategy: useCallback for function props

```tsx
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
<MemoizedChild onClick={handleClick} />
```

### Strategy: Derive minimal data

```tsx
// BAD: Entire user object causes re-render when any field changes
<CallToAction user={user} />

// GOOD: Pass only what's needed
const hasGroups = user.groups !== null;
<CallToAction hasGroups={hasGroups} />
```

---

## Pattern 3: Context Performance Optimization

Context changes re-render ALL consumers. Split contexts to minimize unnecessary re-renders.

### Split static and dynamic values

```tsx
// BAD: Every consumer re-renders when either value changes
const AppContext = createContext<{ theme: string; user: User | null }>({
  theme: 'light',
  user: null,
});

// GOOD: Separate contexts — theme consumers don't re-render on user change
const ThemeContext = createContext<string>('light');
const UserContext = createContext<User | null>(null);
```

### Memoize the context value

```tsx
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState('light');

  // useMemo prevents unnecessary re-renders of consumers
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

### Move state down (lift content up)

```tsx
// BAD: ExpensiveTree re-renders on every color change
function App() {
  const [color, setColor] = useState('red');
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      <ExpensiveTree />
    </div>
  );
}

// GOOD: Extract the state-dependent part
function App() {
  return (
    <ColorPicker>
      <ExpensiveTree />
    </ColorPicker>
  );
}

function ColorPicker({ children }: { children: React.ReactNode }) {
  const [color, setColor] = useState('red');
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      {children} {/* children is a stable reference, does not re-render */}
    </div>
  );
}
```

---

## Pattern 4: Lazy Loading with Preload Hints

Combine `React.lazy` with module preloading for better UX.

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));

// Preload on hover — starts loading before the user clicks
function NavLink() {
  const preload = () => {
    import('./pages/Dashboard');
  };

  return (
    <a href="/dashboard" onMouseEnter={preload} onFocus={preload}>
      Dashboard
    </a>
  );
}
```

---

## Pattern 5: Debounced Input with Deferred Value

Use `useDeferredValue` for non-urgent updates (keeps input responsive).

```tsx
import { useState, useDeferredValue, useMemo } from 'react';

function SearchPage({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  // Expensive filter uses deferred value — input stays responsive
  const results = useMemo(
    () => items.filter((item) => item.includes(deferredQuery)),
    [items, deferredQuery]
  );

  const isStale = query !== deferredQuery;

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        <ResultList results={results} />
      </div>
    </div>
  );
}
```

---

## Pattern 6: Image Optimization

### Lazy loading images

```tsx
function ProductImage({ src, alt }: { src: string; alt: string }) {
  return (
    <img
      src={src}
      alt={alt}
      loading="lazy"        // Browser-native lazy loading
      decoding="async"      // Non-blocking decode
      width={300}           // ALWAYS set dimensions to prevent layout shift
      height={200}
    />
  );
}
```

### Responsive images with srcSet

```tsx
function ResponsiveImage({ baseSrc, alt }: { baseSrc: string; alt: string }) {
  return (
    <img
      src={`${baseSrc}-800.webp`}
      srcSet={`${baseSrc}-400.webp 400w, ${baseSrc}-800.webp 800w, ${baseSrc}-1200.webp 1200w`}
      sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
      alt={alt}
      loading="lazy"
      decoding="async"
      width={800}
      height={600}
    />
  );
}
```

ALWAYS set explicit `width` and `height` to prevent Cumulative Layout Shift (CLS).

---

## Pattern 7: Batch State Updates

React 18+ automatically batches state updates in event handlers, promises, setTimeout, and native event handlers. No manual batching needed.

```tsx
// React 18+: Both updates result in a SINGLE re-render
function handleClick() {
  setCount((c) => c + 1);
  setFlag((f) => !f);
  // React batches these — one render, not two
}

// Also batched in async contexts (new in React 18)
async function handleSubmit() {
  const data = await fetchData();
  setData(data);
  setLoading(false);
  // Single re-render
}
```

NEVER use `flushSync` unless you need the DOM updated before the next line (e.g., measuring after a state change). It breaks batching and hurts performance.
