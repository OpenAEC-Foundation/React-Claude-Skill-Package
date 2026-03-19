# react-core-state -- State Architecture Patterns

## Pattern 1: Context with Split Providers

Split unrelated state into separate contexts. This prevents unrelated re-renders.

```typescript
// ALWAYS split contexts by concern -- NEVER put unrelated state in one context

// auth-context.ts
interface AuthContextType {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = useCallback(async (credentials: Credentials) => {
    const user = await authenticateAPI(credentials);
    setUser(user);
  }, []);

  const logout = useCallback(() => {
    setUser(null);
  }, []);

  // ALWAYS memoize context value to prevent consumer re-renders
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

  return <AuthContext value={value}>{children}</AuthContext>;
}

function useAuth(): AuthContextType {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}

// theme-context.ts -- separate from auth
interface ThemeContextType {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const toggleTheme = useCallback(() => {
    setTheme(prev => (prev === "light" ? "dark" : "light"));
  }, []);

  const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);

  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

### Composing Providers

```typescript
// Compose providers at the app root
function App() {
  return (
    <AuthProvider>
      <ThemeProvider>
        <NotificationProvider>
          <Router />
        </NotificationProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```

---

## Pattern 2: Context + useReducer for Complex Domain State

For state with many transitions, combine Context with `useReducer`.

```typescript
interface AppState {
  notifications: Notification[];
  sidebar: { open: boolean; activeTab: string };
}

type AppAction =
  | { type: "ADD_NOTIFICATION"; notification: Notification }
  | { type: "DISMISS_NOTIFICATION"; id: string }
  | { type: "TOGGLE_SIDEBAR" }
  | { type: "SET_SIDEBAR_TAB"; tab: string };

function appReducer(state: AppState, action: AppAction): AppState {
  switch (action.type) {
    case "ADD_NOTIFICATION":
      return { ...state, notifications: [...state.notifications, action.notification] };
    case "DISMISS_NOTIFICATION":
      return {
        ...state,
        notifications: state.notifications.filter(n => n.id !== action.id),
      };
    case "TOGGLE_SIDEBAR":
      return { ...state, sidebar: { ...state.sidebar, open: !state.sidebar.open } };
    case "SET_SIDEBAR_TAB":
      return { ...state, sidebar: { ...state.sidebar, activeTab: action.tab } };
  }
}

// Split state and dispatch into separate contexts
// This way, components that only dispatch don't re-render on state changes
const AppStateContext = createContext<AppState | null>(null);
const AppDispatchContext = createContext<React.Dispatch<AppAction> | null>(null);

function AppStateProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(appReducer, {
    notifications: [],
    sidebar: { open: false, activeTab: "home" },
  });

  return (
    <AppStateContext value={state}>
      <AppDispatchContext value={dispatch}>
        {children}
      </AppDispatchContext>
    </AppStateContext>
  );
}

// Typed hooks
function useAppState(): AppState {
  const ctx = useContext(AppStateContext);
  if (!ctx) throw new Error("useAppState must be used within AppStateProvider");
  return ctx;
}

function useAppDispatch(): React.Dispatch<AppAction> {
  const ctx = useContext(AppDispatchContext);
  if (!ctx) throw new Error("useAppDispatch must be used within AppStateProvider");
  return ctx;
}
```

---

## Pattern 3: Zustand -- Lightweight External Store

Zustand provides selector-based subscriptions. Components ONLY re-render when their selected slice changes.

```typescript
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";

interface BearStore {
  bears: number;
  fish: number;
  addBear: () => void;
  addFish: () => void;
  reset: () => void;
}

const useBearStore = create<BearStore>()(
  devtools(
    persist(
      (set) => ({
        bears: 0,
        fish: 0,
        addBear: () => set((s) => ({ bears: s.bears + 1 })),
        addFish: () => set((s) => ({ fish: s.fish + 1 })),
        reset: () => set({ bears: 0, fish: 0 }),
      }),
      { name: "bear-storage" } // localStorage key
    )
  )
);

// ALWAYS use selectors -- component only re-renders when bears changes
function BearCount() {
  const bears = useBearStore((s) => s.bears);
  return <span>{bears} bears</span>;
}

// This component does NOT re-render when bears changes
function FishCount() {
  const fish = useBearStore((s) => s.fish);
  return <span>{fish} fish</span>;
}

// Actions can be used outside React components
function resetFromAPI() {
  useBearStore.getState().reset();
}
```

### Zustand with Async Actions

```typescript
interface DataStore {
  data: Item[] | null;
  loading: boolean;
  error: string | null;
  fetchData: () => Promise<void>;
}

const useDataStore = create<DataStore>()((set) => ({
  data: null,
  loading: false,
  error: null,
  fetchData: async () => {
    set({ loading: true, error: null });
    try {
      const data = await fetchAPI<Item[]>("/items");
      set({ data, loading: false });
    } catch (err) {
      set({ error: (err as Error).message, loading: false });
    }
  },
}));
```

---

## Pattern 4: Jotai -- Atomic State

Jotai uses atoms (bottom-up approach). Each atom is an independent piece of state.

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from "jotai";

// Primitive atoms
const countAtom = atom<number>(0);
const nameAtom = atom<string>("React");

// Derived atom (read-only) -- computed from other atoms
const doubleCountAtom = atom<number>((get) => get(countAtom) * 2);

// Writable derived atom
const uppercaseNameAtom = atom(
  (get) => get(nameAtom).toUpperCase(),
  (_get, set, newName: string) => set(nameAtom, newName.toLowerCase())
);

// Async derived atom
const userAtom = atom(async (get) => {
  const id = get(userIdAtom);
  const response = await fetch(`/api/users/${id}`);
  return response.json() as Promise<User>;
});

// Usage in components
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubleCount = useAtomValue(doubleCountAtom); // read-only
  const setName = useSetAtom(nameAtom); // write-only (no re-render on name change)

  return (
    <div>
      <p>{count} (double: {doubleCount})</p>
      <button onClick={() => setCount(prev => prev + 1)}>+1</button>
    </div>
  );
}
```

---

## Pattern 5: Server State with TanStack Query

Server state has different concerns than client state: caching, revalidation, deduplication, background refetching.

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

// Fetch -- automatic caching and deduplication
function useTodos() {
  return useQuery<Todo[]>({
    queryKey: ["todos"],
    queryFn: () => fetch("/api/todos").then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes before refetch
  });
}

// Mutate -- with optimistic update
function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (todo: Todo) =>
      fetch(`/api/todos/${todo.id}`, {
        method: "PATCH",
        body: JSON.stringify({ completed: !todo.completed }),
      }).then(r => r.json()),

    // Optimistic update
    onMutate: async (todo) => {
      await queryClient.cancelQueries({ queryKey: ["todos"] });
      const previous = queryClient.getQueryData<Todo[]>(["todos"]);

      queryClient.setQueryData<Todo[]>(["todos"], (old) =>
        old?.map(t =>
          t.id === todo.id ? { ...t, completed: !t.completed } : t
        )
      );

      return { previous };
    },

    // Rollback on error
    onError: (_err, _todo, context) => {
      if (context?.previous) {
        queryClient.setQueryData(["todos"], context.previous);
      }
    },

    // Refetch after settle
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}

// Usage
function TodoList() {
  const { data: todos, isLoading, error } = useTodos();
  const toggleMutation = useToggleTodo();

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {todos?.map(todo => (
        <li key={todo.id} onClick={() => toggleMutation.mutate(todo)}>
          {todo.completed ? "[x]" : "[ ]"} {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

---

## Pattern 6: State Reset via Key Prop

ALWAYS use the `key` prop to reset component state when the identity changes. This is cleaner than `useEffect` resets.

```typescript
// Parent controls identity via key
function UserProfile({ userId }: { userId: string }) {
  // When userId changes, React unmounts old EditForm and mounts a fresh one
  return <EditForm key={userId} userId={userId} />;
}

// EditForm starts fresh for each userId -- no stale state
function EditForm({ userId }: { userId: string }) {
  const [draft, setDraft] = useState<string>("");
  // draft is always "" when userId changes -- no useEffect needed
  return <input value={draft} onChange={e => setDraft(e.target.value)} />;
}
```

---

## Pattern 7: State Colocation

Keep state as close as possible to where it is used. Only lift when necessary.

```
App
├── Header (uses: theme) -------- theme lives in ThemeContext (shared)
├── Sidebar (uses: isOpen) ------ isOpen lives in Sidebar (local)
├── MainContent
│   ├── SearchBar (uses: query) -- query lifted to MainContent
│   └── ResultsList (uses: query) -- reads from MainContent
└── Footer (no state) ----------- pure presentational
```

**Rule**: If only one component uses a piece of state, it MUST be local. Lift ONLY when a sibling or parent also needs it.
