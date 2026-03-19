---
name: react-impl-testing
description: "Guides React testing with React Testing Library and Vitest including render/screen/fireEvent/waitFor patterns, testing hooks with renderHook, testing async operations, accessibility testing, user-event for realistic interactions, act() for state updates, and snapshot testing. Activates when writing component tests, testing hooks, setting up test infrastructure, or debugging test failures."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript, React Testing Library, and Vitest."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-testing

## Quick Reference

### Installation

```bash
npm install --save-dev vitest @testing-library/react @testing-library/dom \
  @testing-library/jest-dom @testing-library/user-event jsdom
```

### Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    css: true,
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
});
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
```

```jsonc
// tsconfig.json — add to compilerOptions.types
{ "compilerOptions": { "types": ["vitest/globals", "@testing-library/jest-dom"] } }
```

### Query Priority (ALWAYS follow this order)

| Priority | Query | When to Use |
|----------|-------|-------------|
| 1 | `getByRole` | Accessible role — buttons, headings, inputs, links |
| 2 | `getByLabelText` | Form fields with associated `<label>` |
| 3 | `getByPlaceholderText` | Input with placeholder (less accessible) |
| 4 | `getByText` | Non-interactive visible text content |
| 5 | `getByDisplayValue` | Input/textarea with current value |
| 6 | `getByAltText` | Image alt text |
| 7 | `getByTitle` | Title attribute |
| 8 | `getByTestId` | `data-testid` — LAST RESORT ONLY |

### Query Variants

| Variant | No Match | 1 Match | >1 Match | Async? |
|---------|----------|---------|----------|--------|
| `getBy...` | throw | return | throw | No |
| `queryBy...` | `null` | return | throw | No |
| `findBy...` | throw | return | throw | Yes |
| `getAllBy...` | throw | array | array | No |
| `queryAllBy...` | `[]` | array | array | No |
| `findAllBy...` | throw | array | array | Yes |

### Critical Warnings

**NEVER** use Enzyme. It is deprecated and incompatible with React 18+. ALWAYS use React Testing Library.

**NEVER** test implementation details — internal state, instance methods, component internals, or private functions. ALWAYS test visible behavior from the user's perspective.

**NEVER** use `getByTestId` when a semantic query (`getByRole`, `getByLabelText`, `getByText`) is available. `data-testid` is a last resort that provides zero accessibility value.

**ALWAYS** prefer `userEvent` over `fireEvent` for simulating user interactions. `userEvent` fires the full event sequence a real browser would produce.

**ALWAYS** use `screen` for queries instead of destructuring from `render()`. It makes tests more readable and consistent.

**NEVER** wrap RTL interactions in `act()` manually — `render`, `fireEvent`, `userEvent`, and `waitFor` already handle this internally.

---

## Decision Trees

### Which query variant should I use?

```
Need to assert element EXISTS?
├── Yes, and it MUST be there → getBy... (throws on missing)
├── Yes, and I need to WAIT for it → findBy... (async, retries)
└── Maybe not there (asserting absence) → queryBy... (returns null)
```

### fireEvent or userEvent?

```
Simulating user interaction?
├── Typing, clicking, selecting, tabbing → ALWAYS userEvent
├── Custom/synthetic event (resize, scroll) → fireEvent
└── Low-level DOM event testing → fireEvent
```

### Do I need act()?

```
Using RTL utilities (render, fireEvent, userEvent, waitFor)?
├── Yes → act() is handled internally, do NOT add it
└── No (manual state update, direct hook call) → wrap in act()
```

### When to use snapshot testing?

```
Is this a good use case for snapshots?
├── Detecting unintended UI changes → Yes, sparingly
├── Testing specific behavior or logic → No, use assertions
├── Large component tree → No, snapshots become brittle
└── Small, stable output (CSS class, data attribute) → Yes, inline snapshot
```

---

## Patterns

### Pattern 1: Basic Component Test

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Greeting } from './Greeting';

describe('Greeting', () => {
  it('renders the name in a heading', () => {
    render(<Greeting name="Alice" />);

    expect(screen.getByRole('heading', { name: /alice/i })).toBeInTheDocument();
  });
});
```

### Pattern 2: User Interaction with userEvent

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

describe('Counter', () => {
  it('increments count on button click', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: /increment/i }));

    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});
```

### Pattern 3: Async Operations with waitFor

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import { UserList } from './UserList';

describe('UserList', () => {
  it('displays users after loading', async () => {
    render(<UserList />);

    expect(screen.getByText(/loading/i)).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });

    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
});
```

### Pattern 4: Testing Custom Hooks with renderHook

```typescript
import { describe, it, expect } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('starts at initial value', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it('increments the counter', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });
});
```

### Pattern 5: Testing with Context Providers

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';
import { ThemedButton } from './ThemedButton';

function renderWithTheme(ui: React.ReactElement, theme: string = 'light') {
  return render(ui, {
    wrapper: ({ children }) => (
      <ThemeProvider value={theme}>{children}</ThemeProvider>
    ),
  });
}

describe('ThemedButton', () => {
  it('applies dark theme class', () => {
    renderWithTheme(<ThemedButton>Click</ThemedButton>, 'dark');

    expect(screen.getByRole('button', { name: /click/i })).toHaveClass('dark-theme');
  });
});
```

### Pattern 6: API Mocking with vi.fn()

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchForm } from './SearchForm';
import * as api from './api';

vi.mock('./api');

describe('SearchForm', () => {
  beforeEach(() => {
    vi.mocked(api.search).mockResolvedValue([
      { id: '1', title: 'React Testing' },
    ]);
  });

  it('displays search results', async () => {
    const user = userEvent.setup();
    render(<SearchForm />);

    await user.type(screen.getByRole('searchbox'), 'react');
    await user.click(screen.getByRole('button', { name: /search/i }));

    await waitFor(() => {
      expect(screen.getByText('React Testing')).toBeInTheDocument();
    });

    expect(api.search).toHaveBeenCalledWith('react');
  });
});
```

### Pattern 7: Error State Testing

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import { DataLoader } from './DataLoader';
import * as api from './api';

vi.mock('./api');

describe('DataLoader', () => {
  it('displays error message on fetch failure', async () => {
    vi.mocked(api.fetchData).mockRejectedValue(new Error('Network error'));

    render(<DataLoader />);

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/network error/i);
    });
  });
});
```

### Pattern 8: Snapshot Testing (Use Sparingly)

```typescript
import { describe, it, expect } from 'vitest';
import { render } from '@testing-library/react';
import { Badge } from './Badge';

describe('Badge', () => {
  it('matches snapshot for status variant', () => {
    const { asFragment } = render(<Badge status="success">Done</Badge>);
    expect(asFragment()).toMatchSnapshot();
  });

  it('renders correct class via inline snapshot', () => {
    const { container } = render(<Badge status="error">Fail</Badge>);
    expect(container.firstChild).toMatchInlineSnapshot(`
      <span class="badge badge-error">Fail</span>
    `);
  });
});
```

---

## React 18 vs 19 Testing Differences

| Feature | React 18 | React 19 |
|---------|----------|----------|
| `act()` warnings | Frequent in async tests | Improved — fewer false warnings |
| Form actions | Test via `onSubmit` handler | Test via `action` prop with `FormData` |
| `ref` in tests | Use `forwardRef` wrapper | Direct `ref` prop works |
| Suspense | Limited hydration testing | Full Suspense integration |
| `use()` hook | N/A | Test components that read promises in render |

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete test examples for all patterns
- [references/api-table.md](references/api-table.md) -- React Testing Library API reference
- [references/anti-patterns.md](references/anti-patterns.md) -- Common testing mistakes and fixes

### Official Sources

- https://testing-library.com/docs/react-testing-library/intro
- https://testing-library.com/docs/react-testing-library/api
- https://testing-library.com/docs/queries/about
- https://testing-library.com/docs/user-event/intro
- https://vitest.dev/guide/
- https://react.dev/learn/testing
