---
name: react-syntax-context
description: >
  Use when sharing state across components without prop drilling, implementing
  theme/auth/locale providers, or optimizing context performance. Prevents the
  common mistake of putting frequently-changing values in context causing
  unnecessary re-renders. Covers createContext, useContext, Provider pattern,
  TypeScript generics, default values, multiple contexts, performance.
  Keywords: createContext, useContext, Provider, context, prop drilling, theme, share data between components, avoid prop drilling, theme provider, auth context, global data..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-context

## Quick Reference

### Context API Surface

| API | Purpose | Version |
|-----|---------|---------|
| `createContext<T>(defaultValue)` | Create a context object with TypeScript generic | React 18+ |
| `useContext(Context)` | Consume the nearest Provider value | React 18+ |
| `use(Context)` | Consume context inside conditionals/loops | React 19 only |
| `<Context.Provider value={...}>` | Provide value to descendants | React 18 |
| `<Context value={...}>` | Provide value to descendants (no `.Provider`) | React 19 |

### Critical Warnings

**NEVER** create a new object literal directly in the Provider `value` prop without `useMemo` -- this creates a new reference every render and forces ALL consumers to re-render.

**NEVER** use Context for frequently changing values (e.g., mouse position, scroll offset, animation frames) -- use `useSyncExternalStore` or an external state library instead.

**NEVER** put everything in a single global context -- split by concern (theme, auth, locale) to prevent unrelated re-renders.

**ALWAYS** provide a custom hook wrapper (e.g., `useAuth()`) around `useContext` -- this centralizes the missing-provider check and improves API ergonomics.

**ALWAYS** use `useMemo` to stabilize context value objects -- this prevents unnecessary consumer re-renders when the provider's parent re-renders.

---

## Decision Tree: Do You Need Context?

```
Need to share state across components?
├── Only 1-2 levels deep? → Pass props directly (no Context needed)
├── Many levels deep but rarely changes? → Use Context
├── Changes frequently (>1x per second)? → Use external store (Zustand, Jotai, useSyncExternalStore)
├── Server-only data (no interactivity)? → Pass as props from Server Component
└── Complex state with actions? → Context + useReducer
```

---

## Creating Context with TypeScript

### Pattern: Null Default with Type Assertion

```tsx
import { createContext, useContext, type ReactNode } from "react";

// 1. Define the context type
interface AuthContextType {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

// 2. Create with null default -- ALWAYS use this pattern for contexts
//    that REQUIRE a provider
const AuthContext = createContext<AuthContextType | null>(null);

// 3. Custom hook with missing-provider guard
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

**Why null default?** Passing a "real" default value hides bugs where a Provider is missing. The null pattern forces an explicit error at the call site.

### Pattern: Safe Default (No Provider Required)

```tsx
// Use when a sensible default exists and Provider is optional
const ThemeContext = createContext<"light" | "dark">("light");

// No null check needed -- always returns a valid value
function useTheme(): "light" | "dark" {
  return useContext(ThemeContext);
}
```

---

## Provider Patterns

### Custom Provider Component

ALWAYS encapsulate state logic inside a custom Provider component:

```tsx
interface AuthProviderProps {
  children: ReactNode;
}

function AuthProvider({ children }: AuthProviderProps): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  const login = useCallback(async (credentials: Credentials) => {
    const result = await authApi.login(credentials);
    setUser(result.user);
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    authApi.logout();
  }, []);

  // ALWAYS stabilize the value object with useMemo
  const value = useMemo<AuthContextType>(
    () => ({ user, login, logout }),
    [user, login, logout]
  );

  // React 19: <AuthContext value={value}>
  // React 18: <AuthContext.Provider value={value}>
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

### Nested Providers

The closest Provider wins. Inner providers override outer ones:

```tsx
function App(): JSX.Element {
  return (
    <ThemeContext.Provider value="dark">
      <Sidebar />
      <ThemeContext.Provider value="light">
        <MainContent /> {/* reads "light" */}
      </ThemeContext.Provider>
    </ThemeContext.Provider>
  );
}
```

---

## Multiple Contexts: Split by Concern

ALWAYS split unrelated concerns into separate contexts:

```tsx
// GOOD -- independent contexts
const ThemeContext = createContext<ThemeContextType | null>(null);
const AuthContext = createContext<AuthContextType | null>(null);
const LocaleContext = createContext<LocaleContextType | null>(null);

// Compose providers at app root
function AppProviders({ children }: { children: ReactNode }): JSX.Element {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>
          {children}
        </LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```

**Why split?** When `user` changes in a combined context, components that only need `theme` still re-render. Separate contexts prevent this.

---

## Performance: Splitting Read and Write Contexts

For state + dispatch patterns, split into TWO contexts to prevent dispatch-only consumers from re-rendering when state changes:

```tsx
const TodosStateContext = createContext<TodosState | null>(null);
const TodosDispatchContext = createContext<TodosDispatch | null>(null);

function TodosProvider({ children }: { children: ReactNode }): JSX.Element {
  const [state, dispatch] = useReducer(todosReducer, initialState);

  // State changes every update -- only state consumers re-render
  // Dispatch is stable -- dispatch consumers NEVER re-render from state changes
  return (
    <TodosStateContext.Provider value={state}>
      <TodosDispatchContext.Provider value={dispatch}>
        {children}
      </TodosDispatchContext.Provider>
    </TodosStateContext.Provider>
  );
}

// Targeted hooks
function useTodosState(): TodosState {
  const ctx = useContext(TodosStateContext);
  if (ctx === null) throw new Error("useTodosState requires TodosProvider");
  return ctx;
}

function useTodosDispatch(): TodosDispatch {
  const ctx = useContext(TodosDispatchContext);
  if (ctx === null) throw new Error("useTodosDispatch requires TodosProvider");
  return ctx;
}
```

Components that only call `useTodosDispatch()` will NOT re-render when todos state changes.

---

## Context + useReducer

ALWAYS prefer `useReducer` over `useState` when context manages complex state with multiple actions:

```tsx
type TodoAction =
  | { type: "ADD"; text: string }
  | { type: "TOGGLE"; id: number }
  | { type: "DELETE"; id: number };

interface TodosState {
  todos: Todo[];
  filter: "all" | "active" | "completed";
}

type TodosDispatch = React.Dispatch<TodoAction>;

function todosReducer(state: TodosState, action: TodoAction): TodosState {
  switch (action.type) {
    case "ADD":
      return {
        ...state,
        todos: [...state.todos, { id: Date.now(), text: action.text, done: false }],
      };
    case "TOGGLE":
      return {
        ...state,
        todos: state.todos.map((t) =>
          t.id === action.id ? { ...t, done: !t.done } : t
        ),
      };
    case "DELETE":
      return {
        ...state,
        todos: state.todos.filter((t) => t.id !== action.id),
      };
  }
}
```

---

## Value Stabilization with useMemo

```tsx
// BAD -- new object every render, ALL consumers re-render
function ThemeProvider({ children }: { children: ReactNode }): JSX.Element {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// GOOD -- stable reference, consumers re-render only when theme changes
function ThemeProvider({ children }: { children: ReactNode }): JSX.Element {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}
```

---

## React 19: use(Context) and Context as Provider

### use(Context) -- Conditional Context Reading

```tsx
// React 19 ONLY -- use() can be called inside conditionals
function StatusBadge({ showAuth }: { showAuth: boolean }): JSX.Element {
  if (showAuth) {
    // ALLOWED with use() -- FORBIDDEN with useContext()
    const { user } = use(AuthContext);
    return <Badge user={user} />;
  }
  return <GuestBadge />;
}
```

### Context as Provider (No .Provider)

```tsx
// React 19 -- render Context directly
<ThemeContext value={theme}>
  <App />
</ThemeContext>

// React 18 -- must use .Provider
<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>
```

React 19 will deprecate `<Context.Provider>` in a future minor release.

---

## When NOT to Use Context

| Scenario | Why Not Context | Use Instead |
|----------|----------------|-------------|
| Frequently changing values (>1x/sec) | Every change re-renders ALL consumers | `useSyncExternalStore`, Zustand, Jotai |
| Large global state (100+ fields) | Single update triggers widespread re-renders | External state library with selectors |
| Animation values | 60fps updates re-render entire consumer tree | CSS variables, refs, animation libraries |
| Form state across many fields | Each keystroke re-renders all field consumers | React Hook Form, Formik |

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete context patterns with TypeScript
- [references/anti-patterns.md](references/anti-patterns.md) -- Common context mistakes and fixes

### Official Sources

- https://react.dev/reference/react/createContext
- https://react.dev/reference/react/useContext
- https://react.dev/reference/react/use
- https://react.dev/learn/passing-data-deeply-with-context
- https://react.dev/learn/scaling-up-with-reducer-and-context
