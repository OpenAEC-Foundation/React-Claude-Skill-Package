# react-impl-testing — Anti-Patterns

Common testing mistakes, why they are wrong, and the correct alternative.

---

## Anti-Pattern 1: Testing Implementation Details

```typescript
// WRONG: Testing internal state
it('updates internal state', () => {
  const { result } = renderHook(() => useState(0));
  // Directly inspecting or manipulating component internals
});

// WRONG: Testing that a specific function was called internally
it('calls the internal handler', () => {
  const wrapper = shallow(<Counter />); // Enzyme — NEVER use
  wrapper.instance().handleClick();
  expect(wrapper.state('count')).toBe(1);
});

// CORRECT: Test visible behavior from the user's perspective
it('displays incremented count after clicking', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

**WHY**: Implementation details change during refactoring. Tests that depend on internal state break when you refactor without changing behavior. User-facing tests survive refactors and provide real confidence.

---

## Anti-Pattern 2: Using Enzyme

```typescript
// WRONG: Enzyme is deprecated and incompatible with React 18+
import { shallow, mount } from 'enzyme';

const wrapper = shallow(<MyComponent />);
expect(wrapper.find('.my-class')).toHaveLength(1);
wrapper.instance().someMethod(); // accessing component instance
```

**WHY**: Enzyme encourages testing implementation details (shallow rendering, component instances, internal state). It has no official adapter for React 18 or 19. ALWAYS use React Testing Library instead.

---

## Anti-Pattern 3: Using getByTestId as First Choice

```typescript
// WRONG: Using data-testid when semantic queries are available
render(<button data-testid="submit-btn">Submit Order</button>);
screen.getByTestId('submit-btn');

// CORRECT: Use getByRole — it also verifies accessibility
screen.getByRole('button', { name: /submit order/i });
```

**WHY**: `getByRole` validates that the element has the correct accessible role and name. `getByTestId` provides zero accessibility verification. If you can query by role, label, or text, those queries are strictly better.

---

## Anti-Pattern 4: Wrapping Everything in act()

```typescript
// WRONG: Unnecessary act() — RTL already wraps internally
await act(async () => {
  render(<MyComponent />);
});

await act(async () => {
  fireEvent.click(screen.getByRole('button'));
});

await act(async () => {
  await waitFor(() => {
    expect(screen.getByText('Done')).toBeInTheDocument();
  });
});

// CORRECT: RTL methods handle act() internally
render(<MyComponent />);
await userEvent.setup().click(screen.getByRole('button'));
await waitFor(() => {
  expect(screen.getByText('Done')).toBeInTheDocument();
});
```

**WHY**: `render()`, `fireEvent`, `userEvent`, and `waitFor` all wrap their operations in `act()` internally. Adding an outer `act()` is redundant and makes tests harder to read. Only use `act()` when directly calling hook functions from `renderHook`.

---

## Anti-Pattern 5: Using Fixed Delays Instead of waitFor

```typescript
// WRONG: Arbitrary setTimeout delay
it('loads data', async () => {
  render(<DataList />);

  await new Promise((r) => setTimeout(r, 2000)); // fragile!

  expect(screen.getByText('Item 1')).toBeInTheDocument();
});

// CORRECT: waitFor retries until the assertion passes or times out
it('loads data', async () => {
  render(<DataList />);

  await waitFor(() => {
    expect(screen.getByText('Item 1')).toBeInTheDocument();
  });
});

// ALSO CORRECT: findBy queries are waitFor + getBy combined
it('loads data', async () => {
  render(<DataList />);

  expect(await screen.findByText('Item 1')).toBeInTheDocument();
});
```

**WHY**: Fixed delays make tests slow (waiting full duration) and flaky (fails if async operation takes longer than expected). `waitFor` polls at short intervals and resolves as soon as the assertion passes.

---

## Anti-Pattern 6: Not Cleaning Up Mocks

```typescript
// WRONG: Mock leaks between tests
vi.mock('./api');

describe('ComponentA', () => {
  it('test one', () => {
    vi.mocked(api.fetchData).mockResolvedValue({ name: 'Alice' });
    render(<ComponentA />);
    // ...
  });

  it('test two', () => {
    // fetchData still returns Alice! Mock was not reset
    render(<ComponentA />);
    // ...
  });
});

// CORRECT: Reset mocks before each test
describe('ComponentA', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('test one', () => {
    vi.mocked(api.fetchData).mockResolvedValue({ name: 'Alice' });
    // ...
  });

  it('test two', () => {
    vi.mocked(api.fetchData).mockResolvedValue({ name: 'Bob' });
    // ...
  });
});
```

**WHY**: Mock state persists across tests in the same `describe` block. Stale mocks cause tests to pass or fail depending on execution order — a common source of flaky tests.

---

## Anti-Pattern 7: Destructuring Queries from render()

```typescript
// WRONG: Destructuring creates scope confusion
const { getByText, getByRole } = render(<MyComponent />);
getByText('Hello');

// CORRECT: Use screen — single, consistent API surface
render(<MyComponent />);
screen.getByText('Hello');
```

**WHY**: `screen` is always available and does not need to be destructured. It reduces variable noise and makes it clear which DOM tree is being queried. The only exception is `container` and helper methods like `debug()`, `rerender()`, and `unmount()`.

---

## Anti-Pattern 8: Testing CSS Class Names Directly

```typescript
// WRONG: Tied to CSS implementation
expect(screen.getByRole('button')).toHaveClass('btn-primary-active-large');

// CORRECT: Test visible behavior or accessibility
expect(screen.getByRole('button')).toBeEnabled();
expect(screen.getByRole('button')).toBeVisible();
expect(screen.getByRole('button')).toHaveAccessibleName('Submit');
```

**WHY**: CSS class names are implementation details that change during styling refactors. Test what the user sees and experiences — enabled state, visibility, accessible names, text content. Exception: testing a theme or variant system where classes ARE the public API.

---

## Anti-Pattern 9: Large Snapshot Tests

```typescript
// WRONG: Full page snapshot — breaks on any change, hard to review
it('renders correctly', () => {
  const { asFragment } = render(<EntireApp />);
  expect(asFragment()).toMatchSnapshot();
});

// CORRECT: Small, focused inline snapshots for stable output
it('renders badge with correct structure', () => {
  const { container } = render(<Badge variant="success">Done</Badge>);
  expect(container.firstChild).toMatchInlineSnapshot(`
    <span class="badge badge-success">Done</span>
  `);
});

// BETTER: Use specific assertions instead of snapshots
it('renders success badge', () => {
  render(<Badge variant="success">Done</Badge>);
  expect(screen.getByText('Done')).toHaveClass('badge-success');
});
```

**WHY**: Large snapshots are noise — reviewers approve changes without reading them. They break on unrelated changes (a sibling component update, a key change). Use specific assertions for behavior and reserve snapshots for small, stable output only.

---

## Anti-Pattern 10: Not Testing Error States

```typescript
// WRONG: Only testing the happy path
describe('UserProfile', () => {
  it('renders user data', async () => {
    vi.mocked(api.getUser).mockResolvedValue({ name: 'Alice' });
    render(<UserProfile userId="1" />);
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });
  // Missing: error state, loading state, empty state
});

// CORRECT: Test all states
describe('UserProfile', () => {
  it('shows loading indicator', () => {
    vi.mocked(api.getUser).mockReturnValue(new Promise(() => {}));
    render(<UserProfile userId="1" />);
    expect(screen.getByRole('status')).toBeInTheDocument();
  });

  it('renders user data on success', async () => {
    vi.mocked(api.getUser).mockResolvedValue({ name: 'Alice' });
    render(<UserProfile userId="1" />);
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });

  it('shows error message on failure', async () => {
    vi.mocked(api.getUser).mockRejectedValue(new Error('Not found'));
    render(<UserProfile userId="1" />);
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/not found/i);
    });
  });

  it('shows empty state for missing data', async () => {
    vi.mocked(api.getUser).mockResolvedValue(null);
    render(<UserProfile userId="1" />);
    await waitFor(() => {
      expect(screen.getByText(/no user found/i)).toBeInTheDocument();
    });
  });
});
```

**WHY**: Happy-path-only tests give false confidence. Users encounter errors, slow networks, and empty states. ALWAYS test loading, success, error, and empty states for any async component.

---

## Anti-Pattern 11: Querying by DOM Structure

```typescript
// WRONG: Brittle — breaks when HTML structure changes
const { container } = render(<Nav />);
const link = container.querySelector('nav > ul > li:first-child > a');

// CORRECT: Query by accessible role and name
const link = screen.getByRole('link', { name: /home/i });
```

**WHY**: DOM structure is an implementation detail. Changing from `<ul>` to `<nav>` with `<a>` links, or wrapping items in a `<div>`, breaks structure-based selectors. Accessible queries survive structural refactors.

---

## Summary: What to Test and How

| Test This | How |
|-----------|-----|
| User sees text | `screen.getByText()`, `toHaveTextContent()` |
| User clicks button | `userEvent.click()` + assert visible result |
| Form submission | `userEvent.type()` + `userEvent.click()` + assert callback/result |
| Async data loading | `waitFor()` or `findBy` queries |
| Error states | Mock rejection + assert error UI |
| Accessibility | `getByRole()`, `getByLabelText()`, `toHaveAccessibleName()` |
| Hook behavior | `renderHook()` + `act()` for updates |

| NEVER Test This | Why |
|-----------------|-----|
| Internal state values | Implementation detail |
| Component instance methods | Implementation detail |
| CSS class names (usually) | Styling detail |
| DOM structure/nesting | HTML detail |
| Specific rendered HTML | Brittle |
