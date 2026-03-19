# react-impl-testing — API Reference

Complete API reference for React Testing Library, user-event, and jest-dom matchers.

---

## render()

```typescript
import { render } from '@testing-library/react';

function render(
  ui: React.ReactElement,
  options?: {
    container?: Element;
    baseElement?: Element;
    hydrate?: boolean;
    wrapper?: React.ComponentType<{ children: React.ReactNode }>;
    queries?: typeof queries;
  }
): RenderResult;
```

### RenderResult

| Property | Type | Description |
|----------|------|-------------|
| `...queries` | Query functions | All `getBy`, `queryBy`, `findBy` queries bound to the container |
| `container` | `HTMLElement` | The DOM node wrapping the rendered component |
| `baseElement` | `HTMLElement` | The `document.body` or custom base element |
| `debug(el?)` | `() => void` | Logs formatted DOM output to console |
| `rerender(ui)` | `(ui: ReactElement) => void` | Re-render with new props |
| `unmount()` | `() => void` | Unmount the rendered component |
| `asFragment()` | `() => DocumentFragment` | Returns a DocumentFragment for snapshot testing |

---

## screen

```typescript
import { screen } from '@testing-library/react';
```

All query methods are available on `screen` after calling `render()`. ALWAYS use `screen` instead of destructuring queries from `render()`.

---

## Query Methods

### By Role (PREFERRED)

```typescript
screen.getByRole('button');
screen.getByRole('button', { name: /submit/i });
screen.getByRole('heading', { level: 2 });
screen.getByRole('textbox', { name: /email/i });
screen.getByRole('checkbox', { checked: true });
screen.getByRole('combobox', { expanded: true });
screen.getByRole('tab', { selected: true });
screen.getByRole('link', { name: /home/i });
screen.getByRole('alert');
screen.getByRole('dialog');
screen.getByRole('navigation');
screen.getByRole('list');
screen.getByRole('listitem');
screen.getByRole('region', { name: /sidebar/i });
```

### Common ARIA Roles

| HTML Element | Implicit Role |
|-------------|---------------|
| `<button>` | `button` |
| `<a href>` | `link` |
| `<input type="text">` | `textbox` |
| `<input type="checkbox">` | `checkbox` |
| `<input type="radio">` | `radio` |
| `<select>` | `combobox` |
| `<textarea>` | `textbox` |
| `<h1>`-`<h6>` | `heading` |
| `<img>` | `img` |
| `<nav>` | `navigation` |
| `<ul>`, `<ol>` | `list` |
| `<li>` | `listitem` |
| `<table>` | `table` |
| `<tr>` | `row` |
| `<td>` | `cell` |
| `<form>` | `form` (with accessible name) |
| `<dialog>` | `dialog` |

### By Label

```typescript
screen.getByLabelText('Email');
screen.getByLabelText(/email/i);
screen.getByLabelText('Email', { selector: 'input' }); // disambiguate
```

### By Text

```typescript
screen.getByText('Hello World');
screen.getByText(/hello/i);
screen.getByText((content, element) => {
  return element?.tagName === 'SPAN' && content.startsWith('Hello');
});
```

### By Test ID

```typescript
// HTML: <div data-testid="custom-element" />
screen.getByTestId('custom-element');
```

---

## waitFor()

```typescript
import { waitFor } from '@testing-library/react';

await waitFor(
  () => expect(screen.getByText('Loaded')).toBeInTheDocument(),
  {
    timeout: 1000,     // default: 1000ms
    interval: 50,      // default: 50ms (polling interval)
    onTimeout: (error) => error, // custom timeout error
  }
);
```

ALWAYS use `waitFor` for assertions on async operations. NEVER use arbitrary `setTimeout` delays.

---

## waitForElementToBeRemoved()

```typescript
import { waitForElementToBeRemoved } from '@testing-library/react';

await waitForElementToBeRemoved(() => screen.queryByText(/loading/i));
```

---

## within()

```typescript
import { within } from '@testing-library/react';

const section = screen.getByRole('region', { name: /users/i });
const heading = within(section).getByRole('heading');
```

---

## renderHook()

```typescript
import { renderHook } from '@testing-library/react';

const { result, rerender, unmount } = renderHook(
  (props) => useMyHook(props.value),
  {
    initialProps: { value: 'initial' },
    wrapper: MyContextProvider,
  }
);

// Access current return value
result.current;

// Re-render with new props
rerender({ value: 'updated' });

// Unmount the hook
unmount();
```

---

## userEvent

```typescript
import userEvent from '@testing-library/user-event';

const user = userEvent.setup();
```

### Methods

| Method | Description |
|--------|-------------|
| `user.click(element)` | Click an element (focus, pointerdown, mousedown, pointerup, mouseup, click) |
| `user.dblClick(element)` | Double-click |
| `user.tripleClick(element)` | Triple-click (selects text) |
| `user.type(element, text)` | Type text character by character (focus, keydown, keypress, input, keyup per char) |
| `user.clear(element)` | Clear an input/textarea |
| `user.selectOptions(select, values)` | Select option(s) in a `<select>` |
| `user.deselectOptions(select, values)` | Deselect option(s) in a multi-select |
| `user.tab()` | Press Tab key (moves focus) |
| `user.tab({ shift: true })` | Press Shift+Tab |
| `user.keyboard('{Enter}')` | Press specific keys |
| `user.keyboard('{Shift>}A{/Shift}')` | Key combinations |
| `user.hover(element)` | Hover over element |
| `user.unhover(element)` | Move cursor away |
| `user.upload(input, file)` | Upload a file |
| `user.paste(text)` | Paste from clipboard |
| `user.copy()` | Copy selection |
| `user.cut()` | Cut selection |

### Keyboard Key Syntax

| Syntax | Key |
|--------|-----|
| `{Enter}` | Enter |
| `{Escape}` | Escape |
| `{Tab}` | Tab |
| `{Backspace}` | Backspace |
| `{Delete}` | Delete |
| `{ArrowUp}` | Arrow Up |
| `{ArrowDown}` | Arrow Down |
| `{ArrowLeft}` | Arrow Left |
| `{ArrowRight}` | Arrow Right |
| `{Home}` | Home |
| `{End}` | End |
| `{Space}` | Space |
| `{Shift>}` | Hold Shift |
| `{/Shift}` | Release Shift |
| `{Control>}` | Hold Ctrl |
| `{Meta>}` | Hold Meta (Cmd) |

---

## jest-dom Matchers

```typescript
import '@testing-library/jest-dom/vitest';
```

| Matcher | Description |
|---------|-------------|
| `toBeInTheDocument()` | Element exists in the DOM |
| `toBeVisible()` | Element is visible (not hidden by CSS) |
| `toBeEnabled()` | Element is not disabled |
| `toBeDisabled()` | Element has `disabled` attribute |
| `toBeChecked()` | Checkbox/radio is checked |
| `toHaveTextContent(text)` | Element contains text |
| `toHaveValue(value)` | Input/textarea has value |
| `toHaveDisplayValue(value)` | Select/input displays value |
| `toHaveAttribute(attr, value?)` | Element has attribute |
| `toHaveClass(...classes)` | Element has CSS class(es) |
| `toHaveStyle(css)` | Element has inline style |
| `toHaveFocus()` | Element currently has focus |
| `toBeRequired()` | Form element is required |
| `toBeInvalid()` | Form element is invalid |
| `toBeValid()` | Form element is valid |
| `toBeEmptyDOMElement()` | Element has no content |
| `toContainElement(element)` | Element contains another element |
| `toContainHTML(html)` | Element contains HTML string |
| `toHaveAccessibleName(name)` | Element has accessible name |
| `toHaveAccessibleDescription(desc)` | Element has accessible description |
| `toHaveErrorMessage(message)` | Element has error message (via `aria-errormessage`) |
| `toHaveFormValues(values)` | Form contains expected values |

---

## Vitest API (Testing Framework)

### Test Structure

```typescript
import { describe, it, expect, vi, beforeEach, afterEach, beforeAll, afterAll } from 'vitest';

describe('ComponentName', () => {
  beforeEach(() => { /* runs before each test */ });
  afterEach(() => { /* runs after each test */ });

  it('does something', () => {
    expect(value).toBe(expected);
  });

  it.skip('skipped test', () => { /* not run */ });
  it.only('only this test runs', () => { /* focus */ });
  it.todo('implement later');
});
```

### Mocking

```typescript
// Mock a module
vi.mock('./api');

// Mock a specific export
vi.mock('./utils', () => ({
  formatDate: vi.fn(() => '2024-01-01'),
}));

// Create a mock function
const mockFn = vi.fn();
const mockFnWithReturn = vi.fn().mockReturnValue(42);
const mockFnAsync = vi.fn().mockResolvedValue({ data: [] });
const mockFnReject = vi.fn().mockRejectedValue(new Error('fail'));

// Spy on an object method
const spy = vi.spyOn(console, 'error').mockImplementation(() => {});

// Timer mocks
vi.useFakeTimers();
vi.advanceTimersByTime(1000);
vi.runAllTimers();
vi.useRealTimers();

// Reset/restore
vi.clearAllMocks();   // clear call history
vi.resetAllMocks();   // clear history + implementations
vi.restoreAllMocks(); // restore original implementations
```
