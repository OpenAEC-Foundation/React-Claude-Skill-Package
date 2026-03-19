# react-impl-testing — Examples

Complete test examples for all common React testing patterns with React Testing Library and Vitest.

---

## Example 1: Form Submission Test

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits email and password', async () => {
    const handleSubmit = vi.fn();
    const user = userEvent.setup();
    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'user@example.com');
    await user.type(screen.getByLabelText(/password/i), 'secret123');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(handleSubmit).toHaveBeenCalledOnce();
    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'secret123',
    });
  });

  it('shows validation error for empty email', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(screen.getByRole('alert')).toHaveTextContent(/email is required/i);
  });

  it('disables submit button while submitting', async () => {
    const handleSubmit = vi.fn(() => new Promise((r) => setTimeout(r, 100)));
    const user = userEvent.setup();
    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'user@example.com');
    await user.type(screen.getByLabelText(/password/i), 'secret123');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(screen.getByRole('button', { name: /log in/i })).toBeDisabled();
  });
});
```

---

## Example 2: List with Loading and Empty States

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor, within } from '@testing-library/react';
import { TodoList } from './TodoList';
import * as api from './api';

vi.mock('./api');

describe('TodoList', () => {
  it('shows loading spinner initially', () => {
    vi.mocked(api.fetchTodos).mockReturnValue(new Promise(() => {})); // never resolves
    render(<TodoList />);

    expect(screen.getByRole('status')).toBeInTheDocument();
  });

  it('renders todo items after loading', async () => {
    vi.mocked(api.fetchTodos).mockResolvedValue([
      { id: '1', text: 'Buy milk', done: false },
      { id: '2', text: 'Write tests', done: true },
    ]);
    render(<TodoList />);

    await waitFor(() => {
      expect(screen.getByText('Buy milk')).toBeInTheDocument();
    });

    const items = screen.getAllByRole('listitem');
    expect(items).toHaveLength(2);
  });

  it('shows empty message when no todos exist', async () => {
    vi.mocked(api.fetchTodos).mockResolvedValue([]);
    render(<TodoList />);

    await waitFor(() => {
      expect(screen.getByText(/no todos yet/i)).toBeInTheDocument();
    });
  });

  it('shows error message on fetch failure', async () => {
    vi.mocked(api.fetchTodos).mockRejectedValue(new Error('Server down'));
    render(<TodoList />);

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/server down/i);
    });
  });
});
```

---

## Example 3: Component with Context Provider

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { AuthProvider, useAuth } from './AuthContext';
import { UserMenu } from './UserMenu';

// Reusable wrapper for auth context
function renderWithAuth(ui: React.ReactElement, authState = { user: null }) {
  return render(ui, {
    wrapper: ({ children }) => (
      <AuthProvider initialState={authState}>{children}</AuthProvider>
    ),
  });
}

describe('UserMenu', () => {
  it('shows login button when not authenticated', () => {
    renderWithAuth(<UserMenu />);

    expect(screen.getByRole('button', { name: /log in/i })).toBeInTheDocument();
    expect(screen.queryByText(/welcome/i)).not.toBeInTheDocument();
  });

  it('shows username when authenticated', () => {
    renderWithAuth(<UserMenu />, { user: { name: 'Alice', email: 'alice@test.com' } });

    expect(screen.getByText(/welcome, alice/i)).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: /log in/i })).not.toBeInTheDocument();
  });
});
```

---

## Example 4: Testing Custom Hooks

```typescript
import { describe, it, expect, vi } from 'vitest';
import { renderHook, act, waitFor } from '@testing-library/react';
import { useDebounce } from './useDebounce';
import { useLocalStorage } from './useLocalStorage';

describe('useDebounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 500));
    expect(result.current).toBe('hello');
  });

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'hello', delay: 500 } },
    );

    rerender({ value: 'world', delay: 500 });
    expect(result.current).toBe('hello'); // not yet updated

    act(() => {
      vi.advanceTimersByTime(500);
    });

    expect(result.current).toBe('world');
  });
});

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('returns initial value when localStorage is empty', () => {
    const { result } = renderHook(() => useLocalStorage('theme', 'light'));
    expect(result.current[0]).toBe('light');
  });

  it('persists value to localStorage', () => {
    const { result } = renderHook(() => useLocalStorage('theme', 'light'));

    act(() => {
      result.current[1]('dark');
    });

    expect(result.current[0]).toBe('dark');
    expect(localStorage.getItem('theme')).toBe('"dark"');
  });

  it('reads existing value from localStorage', () => {
    localStorage.setItem('theme', '"dark"');
    const { result } = renderHook(() => useLocalStorage('theme', 'light'));
    expect(result.current[0]).toBe('dark');
  });
});
```

---

## Example 5: Keyboard Navigation and Accessibility

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Dropdown } from './Dropdown';

describe('Dropdown', () => {
  const options = ['Apple', 'Banana', 'Cherry'];

  it('opens menu on Enter key', async () => {
    const user = userEvent.setup();
    render(<Dropdown options={options} label="Fruit" />);

    const trigger = screen.getByRole('combobox', { name: /fruit/i });
    await user.tab(); // focus the trigger
    await user.keyboard('{Enter}');

    expect(screen.getByRole('listbox')).toBeInTheDocument();
    expect(screen.getAllByRole('option')).toHaveLength(3);
  });

  it('navigates options with arrow keys', async () => {
    const user = userEvent.setup();
    render(<Dropdown options={options} label="Fruit" />);

    await user.tab();
    await user.keyboard('{Enter}');
    await user.keyboard('{ArrowDown}');
    await user.keyboard('{ArrowDown}');

    expect(screen.getByRole('option', { name: 'Banana' })).toHaveAttribute(
      'aria-selected',
      'true',
    );
  });

  it('selects option on Enter', async () => {
    const onSelect = vi.fn();
    const user = userEvent.setup();
    render(<Dropdown options={options} label="Fruit" onSelect={onSelect} />);

    await user.tab();
    await user.keyboard('{Enter}');
    await user.keyboard('{ArrowDown}');
    await user.keyboard('{Enter}');

    expect(onSelect).toHaveBeenCalledWith('Apple');
  });
});
```

---

## Example 6: Testing with Timer Mocks

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { render, screen, act } from '@testing-library/react';
import { AutoSave } from './AutoSave';

describe('AutoSave', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('saves content after 2 seconds of inactivity', () => {
    const onSave = vi.fn();
    render(<AutoSave content="Draft text" onSave={onSave} />);

    act(() => {
      vi.advanceTimersByTime(2000);
    });

    expect(onSave).toHaveBeenCalledWith('Draft text');
  });

  it('resets timer on content change', () => {
    const onSave = vi.fn();
    const { rerender } = render(<AutoSave content="First" onSave={onSave} />);

    act(() => {
      vi.advanceTimersByTime(1500);
    });

    rerender(<AutoSave content="Second" onSave={onSave} />);

    act(() => {
      vi.advanceTimersByTime(1500);
    });

    expect(onSave).not.toHaveBeenCalled(); // timer was reset

    act(() => {
      vi.advanceTimersByTime(500);
    });

    expect(onSave).toHaveBeenCalledWith('Second');
  });
});
```

---

## Example 7: Testing with Router

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { Navigation } from './Navigation';
import { HomePage } from './HomePage';
import { AboutPage } from './AboutPage';

function renderWithRouter(initialRoute: string = '/') {
  return render(
    <MemoryRouter initialEntries={[initialRoute]}>
      <Navigation />
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/about" element={<AboutPage />} />
      </Routes>
    </MemoryRouter>,
  );
}

describe('Navigation', () => {
  it('renders home page by default', () => {
    renderWithRouter('/');
    expect(screen.getByRole('heading', { name: /home/i })).toBeInTheDocument();
  });

  it('navigates to about page on link click', async () => {
    const user = userEvent.setup();
    renderWithRouter('/');

    await user.click(screen.getByRole('link', { name: /about/i }));

    expect(screen.getByRole('heading', { name: /about/i })).toBeInTheDocument();
  });
});
```

---

## Example 8: Testing React 19 Form Actions

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { FeedbackForm } from './FeedbackForm';

// React 19 form action component
// function FeedbackForm() {
//   async function submitFeedback(formData: FormData) { ... }
//   return <form action={submitFeedback}>...</form>;
// }

describe('FeedbackForm (React 19)', () => {
  it('submits feedback and shows success message', async () => {
    const user = userEvent.setup();
    render(<FeedbackForm />);

    await user.type(screen.getByLabelText(/message/i), 'Great product!');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(screen.getByText(/thank you/i)).toBeInTheDocument();
    });
  });
});
```

---

## Example 9: within() for Scoped Queries

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen, within } from '@testing-library/react';
import { Dashboard } from './Dashboard';

describe('Dashboard', () => {
  it('shows different data in each section', () => {
    render(<Dashboard />);

    const revenueSection = within(screen.getByRole('region', { name: /revenue/i }));
    const usersSection = within(screen.getByRole('region', { name: /users/i }));

    expect(revenueSection.getByText('$12,500')).toBeInTheDocument();
    expect(usersSection.getByText('1,234')).toBeInTheDocument();
  });
});
```
