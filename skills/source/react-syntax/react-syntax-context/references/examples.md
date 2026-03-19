# Context API Examples

> Complete, copy-paste-ready patterns for React Context with TypeScript.

---

## Example 1: Theme Context (Simple Toggle)

```tsx
import {
  createContext,
  useCallback,
  useContext,
  useMemo,
  useState,
  type ReactNode,
} from "react";

// Types
type Theme = "light" | "dark";

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
}

// Context
const ThemeContext = createContext<ThemeContextType | null>(null);

// Custom hook
function useTheme(): ThemeContextType {
  const context = useContext(ThemeContext);
  if (context === null) {
    throw new Error("useTheme must be used within a ThemeProvider");
  }
  return context;
}

// Provider
function ThemeProvider({ children }: { children: ReactNode }): JSX.Element {
  const [theme, setTheme] = useState<Theme>("light");

  const toggleTheme = useCallback(() => {
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  }, []);

  const value = useMemo<ThemeContextType>(
    () => ({ theme, toggleTheme }),
    [theme, toggleTheme]
  );

  return (
    <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
  );
}

// Consumer component
function ThemeToggle(): JSX.Element {
  const { theme, toggleTheme } = useTheme();
  return (
    <button onClick={toggleTheme}>
      Current theme: {theme}
    </button>
  );
}

// App
function App(): JSX.Element {
  return (
    <ThemeProvider>
      <ThemeToggle />
    </ThemeProvider>
  );
}
```

---

## Example 2: Auth Context (Async Operations)

```tsx
import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
  type ReactNode,
} from "react";

// Types
interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

// Context
const AuthContext = createContext<AuthContextType | null>(null);

// Custom hook
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}

// Provider
function AuthProvider({ children }: { children: ReactNode }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState<boolean>(true);

  // Check session on mount
  useEffect(() => {
    let ignore = false;
    async function checkSession(): Promise<void> {
      try {
        const session = await api.getSession();
        if (!ignore) {
          setUser(session.user);
        }
      } catch {
        if (!ignore) {
          setUser(null);
        }
      } finally {
        if (!ignore) {
          setIsLoading(false);
        }
      }
    }
    checkSession();
    return () => {
      ignore = true;
    };
  }, []);

  const login = useCallback(async (email: string, password: string) => {
    const result = await api.login(email, password);
    setUser(result.user);
  }, []);

  const logout = useCallback(async () => {
    await api.logout();
    setUser(null);
  }, []);

  const value = useMemo<AuthContextType>(
    () => ({ user, isLoading, login, logout }),
    [user, isLoading, login, logout]
  );

  return (
    <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
  );
}

// Guard component
function RequireAuth({ children }: { children: ReactNode }): JSX.Element {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <LoadingSpinner />;
  }

  if (user === null) {
    return <Navigate to="/login" />;
  }

  return <>{children}</>;
}
```

---

## Example 3: Context + useReducer (Todo App)

```tsx
import {
  createContext,
  useContext,
  useMemo,
  useReducer,
  type Dispatch,
  type ReactNode,
} from "react";

// Types
interface Todo {
  id: number;
  text: string;
  done: boolean;
}

interface TodosState {
  todos: Todo[];
}

type TodoAction =
  | { type: "ADD"; text: string }
  | { type: "TOGGLE"; id: number }
  | { type: "DELETE"; id: number }
  | { type: "CLEAR_COMPLETED" };

type TodosDispatch = Dispatch<TodoAction>;

// Reducer
function todosReducer(state: TodosState, action: TodoAction): TodosState {
  switch (action.type) {
    case "ADD":
      return {
        ...state,
        todos: [
          ...state.todos,
          { id: Date.now(), text: action.text, done: false },
        ],
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
    case "CLEAR_COMPLETED":
      return {
        ...state,
        todos: state.todos.filter((t) => !t.done),
      };
  }
}

const initialState: TodosState = { todos: [] };

// Split contexts for performance
const TodosStateContext = createContext<TodosState | null>(null);
const TodosDispatchContext = createContext<TodosDispatch | null>(null);

// Custom hooks
function useTodosState(): TodosState {
  const context = useContext(TodosStateContext);
  if (context === null) {
    throw new Error("useTodosState must be used within a TodosProvider");
  }
  return context;
}

function useTodosDispatch(): TodosDispatch {
  const context = useContext(TodosDispatchContext);
  if (context === null) {
    throw new Error("useTodosDispatch must be used within a TodosProvider");
  }
  return context;
}

// Provider
function TodosProvider({ children }: { children: ReactNode }): JSX.Element {
  const [state, dispatch] = useReducer(todosReducer, initialState);

  return (
    <TodosStateContext.Provider value={state}>
      <TodosDispatchContext.Provider value={dispatch}>
        {children}
      </TodosDispatchContext.Provider>
    </TodosStateContext.Provider>
  );
}

// Consumer: reads state (re-renders when todos change)
function TodoList(): JSX.Element {
  const { todos } = useTodosState();
  return (
    <ul>
      {todos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

// Consumer: reads dispatch only (NEVER re-renders from state changes)
function AddTodoForm(): JSX.Element {
  const dispatch = useTodosDispatch();
  const [text, setText] = useState("");

  function handleSubmit(e: React.FormEvent): void {
    e.preventDefault();
    if (text.trim()) {
      dispatch({ type: "ADD", text: text.trim() });
      setText("");
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button type="submit">Add</button>
    </form>
  );
}
```

---

## Example 4: Composing Multiple Providers

```tsx
// Pattern: Provider composer to avoid nesting hell
interface ProviderProps {
  children: ReactNode;
}

type ProviderComponent = React.ComponentType<ProviderProps>;

function ComposeProviders({
  providers,
  children,
}: {
  providers: ProviderComponent[];
  children: ReactNode;
}): JSX.Element {
  return providers.reduceRight<JSX.Element>(
    (acc, Provider) => <Provider>{acc}</Provider>,
    <>{children}</>
  );
}

// Usage
function App(): JSX.Element {
  return (
    <ComposeProviders
      providers={[AuthProvider, ThemeProvider, LocaleProvider, TodosProvider]}
    >
      <MainLayout />
    </ComposeProviders>
  );
}
```

---

## Example 5: React 19 -- use(Context) in Conditionals

```tsx
import { use, createContext } from "react";

const FeatureFlagContext = createContext<Record<string, boolean>>({});

// React 19 ONLY -- use() can be called conditionally
function FeatureGate({
  flag,
  children,
  fallback,
}: {
  flag: string;
  children: ReactNode;
  fallback?: ReactNode;
}): JSX.Element {
  // ALLOWED: use() inside a conditional
  const flags = use(FeatureFlagContext);

  if (flags[flag]) {
    return <>{children}</>;
  }

  return <>{fallback ?? null}</>;
}

// With useContext this would require unconditional call:
function FeatureGateReact18({
  flag,
  children,
  fallback,
}: {
  flag: string;
  children: ReactNode;
  fallback?: ReactNode;
}): JSX.Element {
  // useContext MUST be called unconditionally
  const flags = useContext(FeatureFlagContext);

  if (flags[flag]) {
    return <>{children}</>;
  }

  return <>{fallback ?? null}</>;
}
```

---

## Example 6: React 19 -- Context as Provider

```tsx
// React 19: Context renders directly as a provider
function App(): JSX.Element {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  // No .Provider needed in React 19
  return (
    <ThemeContext value={value}>
      <Page />
    </ThemeContext>
  );
}

// React 18: must use .Provider
function AppReact18(): JSX.Element {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      <Page />
    </ThemeContext.Provider>
  );
}
```

---

## Example 7: Locale Context with Nested Override

```tsx
interface LocaleContextType {
  locale: string;
  t: (key: string) => string;
}

const LocaleContext = createContext<LocaleContextType | null>(null);

function useLocale(): LocaleContextType {
  const context = useContext(LocaleContext);
  if (context === null) {
    throw new Error("useLocale must be used within a LocaleProvider");
  }
  return context;
}

function LocaleProvider({
  locale,
  children,
}: {
  locale: string;
  children: ReactNode;
}): JSX.Element {
  const translations = useTranslations(locale);

  const value = useMemo<LocaleContextType>(
    () => ({
      locale,
      t: (key: string) => translations[key] ?? key,
    }),
    [locale, translations]
  );

  return (
    <LocaleContext.Provider value={value}>{children}</LocaleContext.Provider>
  );
}

// Nested override: admin panel uses English regardless of app locale
function App(): JSX.Element {
  return (
    <LocaleProvider locale="nl">
      <MainContent />
      <LocaleProvider locale="en">
        <AdminPanel /> {/* Always English */}
      </LocaleProvider>
    </LocaleProvider>
  );
}
```
