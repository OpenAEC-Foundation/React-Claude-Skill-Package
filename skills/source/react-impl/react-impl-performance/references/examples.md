# react-impl-performance — Examples

## Example 1: Optimizing a Filterable Product List

### Before (unoptimized)

```tsx
interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
}

function ProductPage({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  // PROBLEM: Recalculates on EVERY render (even when query/sortBy unchanged)
  const filtered = products
    .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
    .sort((a, b) => (sortBy === 'name' ? a.name.localeCompare(b.name) : a.price - b.price));

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={() => setSortBy('name')}>Sort by Name</button>
      <button onClick={() => setSortBy('price')}>Sort by Price</button>
      {/* PROBLEM: New function created every render, defeats any child memoization */}
      <ProductList items={filtered} onSelect={(id) => console.log(id)} />
    </div>
  );
}

function ProductList({ items, onSelect }: { items: Product[]; onSelect: (id: string) => void }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} onClick={() => onSelect(item.id)}>
          {item.name} — ${item.price}
        </li>
      ))}
    </ul>
  );
}
```

### After (optimized with React 18 manual memoization)

```tsx
import { useState, useMemo, useCallback, memo } from 'react';

function ProductPage({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  // useMemo: only recalculate when products, query, or sortBy change
  const filtered = useMemo(() => {
    return products
      .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
      .sort((a, b) => (sortBy === 'name' ? a.name.localeCompare(b.name) : a.price - b.price));
  }, [products, query, sortBy]);

  // useCallback: stable reference for memoized child
  const handleSelect = useCallback((id: string) => {
    console.log(id);
  }, []);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={() => setSortBy('name')}>Sort by Name</button>
      <button onClick={() => setSortBy('price')}>Sort by Price</button>
      <ProductList items={filtered} onSelect={handleSelect} />
    </div>
  );
}

// memo: skip re-render when items and onSelect are referentially equal
const ProductList = memo<{ items: Product[]; onSelect: (id: string) => void }>(
  function ProductList({ items, onSelect }) {
    return (
      <ul>
        {items.map((item) => (
          <li key={item.id} onClick={() => onSelect(item.id)}>
            {item.name} — ${item.price}
          </li>
        ))}
      </ul>
    );
  }
);
```

### After (React 19 with React Compiler)

```tsx
// No manual memo/useMemo/useCallback needed — the compiler handles it automatically.

function ProductPage({ products }: { products: Product[] }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  const filtered = products
    .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
    .sort((a, b) => (sortBy === 'name' ? a.name.localeCompare(b.name) : a.price - b.price));

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={() => setSortBy('name')}>Sort by Name</button>
      <button onClick={() => setSortBy('price')}>Sort by Price</button>
      <ProductList items={filtered} onSelect={(id) => console.log(id)} />
    </div>
  );
}

function ProductList({ items, onSelect }: { items: Product[]; onSelect: (id: string) => void }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} onClick={() => onSelect(item.id)}>
          {item.name} — ${item.price}
        </li>
      ))}
    </ul>
  );
}
```

---

## Example 2: Profiler Usage for Measuring Before Optimizing

```tsx
import { Profiler, useState } from 'react';

interface RenderLog {
  id: string;
  phase: string;
  actualDuration: number;
  baseDuration: number;
}

const renderLogs: RenderLog[] = [];

function onRender(
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
): void {
  renderLogs.push({ id, phase, actualDuration, baseDuration });

  if (actualDuration > 16) {
    console.warn(
      `[Perf] ${id} took ${actualDuration.toFixed(1)}ms (budget: 16ms for 60fps)`
    );
  }
}

function App() {
  return (
    <Profiler id="Sidebar" onRender={onRender}>
      <Sidebar />
    </Profiler>
  );
}
```

---

## Example 3: Route-Based Code Splitting

```tsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// ALWAYS declare lazy components at module level
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="/admin" element={<AdminPanel />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

function PageSkeleton() {
  return <div className="skeleton-page" aria-busy="true">Loading...</div>;
}
```

---

## Example 4: Virtualizing a Large List

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Row {
  id: string;
  name: string;
  email: string;
}

function VirtualTable({ rows }: { rows: Row[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5, // render 5 extra items above/below viewport for smooth scrolling
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) => {
          const row = rows[virtualRow.index];
          return (
            <div
              key={row.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
                display: 'flex',
                alignItems: 'center',
                borderBottom: '1px solid #eee',
              }}
            >
              <span style={{ flex: 1 }}>{row.name}</span>
              <span style={{ flex: 1 }}>{row.email}</span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## Example 5: Measuring and Comparing With/Without Memo

```tsx
import { memo, useMemo, useCallback, Profiler, useState } from 'react';

// Step 1: Add Profiler to measure baseline
function App() {
  const [count, setCount] = useState(0);
  const [items] = useState(() => Array.from({ length: 5000 }, (_, i) => `Item ${i}`));

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>
        Re-render parent (count: {count})
      </button>

      <Profiler id="ItemList" onRender={(id, phase, actual, base) => {
        console.log(`${id}: actual=${actual.toFixed(2)}ms, base=${base.toFixed(2)}ms`);
      }}>
        <ItemList items={items} />
      </Profiler>
    </div>
  );
}

// Step 2: Without memo — re-renders on every parent state change
function ItemList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i}>{item}</li>
      ))}
    </ul>
  );
}

// Step 3: With memo — skips re-render because items reference is stable
const ItemList = memo<{ items: string[] }>(function ItemList({ items }) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i}>{item}</li>
      ))}
    </ul>
  );
});

// Compare Profiler output: actualDuration drops to ~0ms with memo
// because the component is skipped entirely.
```

---

## Example 6: Feature-Based Code Splitting with Dynamic Import

```tsx
import { lazy, Suspense, useState } from 'react';

// Heavy chart library loaded only when user clicks "Show Chart"
const Chart = lazy(() => import('./components/Chart'));

function AnalyticsDashboard({ data }: { data: DataPoint[] }) {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <h2>Analytics</h2>
      <table>{/* lightweight table always loaded */}</table>

      <button onClick={() => setShowChart(true)}>Show Chart</button>

      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <Chart data={data} />
        </Suspense>
      )}
    </div>
  );
}
```
