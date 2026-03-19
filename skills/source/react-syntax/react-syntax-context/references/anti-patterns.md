# Context Anti-Patterns

> Common mistakes when using React Context, with explanations and fixes.

---

## Anti-Pattern 1: Unstable Value Object

**Problem**: Creating a new object in the Provider render body causes ALL consumers to re-render on every parent render, even when the actual values have not changed.

```tsx
// BAD -- new object reference every render
function UserProvider({ children }: { children: ReactNode }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
}
```

**Why it breaks**: React compares context values with `Object.is`. A new object literal `{ user, setUser }` creates a new reference every render, so `Object.is(prevValue, nextValue)` returns `false` even when `user` has not changed.

**Fix**: ALWAYS stabilize the value with `useMemo`:

```tsx
// GOOD -- stable reference, re-renders only when user changes
function UserProvider({ children }: { children: ReactNode }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  const value = useMemo(() => ({ user, setUser }), [user]);

  return (
    <UserContext.Provider value={value}>{children}</UserContext.Provider>
  );
}
```

---

## Anti-Pattern 2: Missing Provider Guard

**Problem**: Using `useContext` without checking for a missing Provider returns the `defaultValue`, which silently hides bugs.

```tsx
// BAD -- returns null silently, crashes later with confusing error
const AuthContext = createContext<AuthContextType | null>(null);

function ProfilePage(): JSX.Element {
  const auth = useContext(AuthContext);
  // auth is null if Provider is missing -- crashes on auth.user access
  return <span>{auth.user.name}</span>;
}
```

**Fix**: ALWAYS create a custom hook that throws a descriptive error:

```tsx
// GOOD -- fails fast with clear message
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}

function ProfilePage(): JSX.Element {
  const { user } = useAuth(); // throws if AuthProvider is missing
  return <span>{user?.name}</span>;
}
```

---

## Anti-Pattern 3: Single Giant Context

**Problem**: Putting all application state into one context causes every consumer to re-render when ANY value changes.

```tsx
// BAD -- theme change re-renders auth consumers, locale consumers, etc.
interface AppContextType {
  user: User | null;
  theme: Theme;
  locale: string;
  notifications: Notification[];
  settings: Settings;
}

const AppContext = createContext<AppContextType | null>(null);
```

**Why it breaks**: When `notifications` updates, components that only read `theme` still re-render because the context value reference changed.

**Fix**: ALWAYS split contexts by concern:

```tsx
// GOOD -- independent update cycles
const AuthContext = createContext<AuthContextType | null>(null);
const ThemeContext = createContext<ThemeContextType | null>(null);
const LocaleContext = createContext<LocaleContextType | null>(null);
const NotificationContext = createContext<NotificationContextType | null>(null);
```

---

## Anti-Pattern 4: Context for Frequently Changing Values

**Problem**: Using Context for values that change many times per second (mouse position, scroll, animation) causes the entire consumer subtree to re-render at that frequency.

```tsx
// BAD -- re-renders ALL consumers 60 times per second
const MouseContext = createContext<{ x: number; y: number }>({ x: 0, y: 0 });

function MouseTracker({ children }: { children: ReactNode }): JSX.Element {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    function handleMove(e: MouseEvent): void {
      setPos({ x: e.clientX, y: e.clientY }); // triggers re-render cascade
    }
    window.addEventListener("mousemove", handleMove);
    return () => window.removeEventListener("mousemove", handleMove);
  }, []);

  return (
    <MouseContext.Provider value={pos}>{children}</MouseContext.Provider>
  );
}
```

**Fix**: Use `useSyncExternalStore` or an external store with selectors:

```tsx
// GOOD -- only subscribers to specific values re-render
import { useSyncExternalStore } from "react";

const mouseStore = {
  position: { x: 0, y: 0 },
  listeners: new Set<() => void>(),
  subscribe(listener: () => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  },
  getSnapshot(): { x: number; y: number } {
    return this.position;
  },
};

window.addEventListener("mousemove", (e) => {
  mouseStore.position = { x: e.clientX, y: e.clientY };
  mouseStore.listeners.forEach((l) => l());
});

function useMousePosition(): { x: number; y: number } {
  return useSyncExternalStore(
    mouseStore.subscribe.bind(mouseStore),
    mouseStore.getSnapshot.bind(mouseStore)
  );
}
```

---

## Anti-Pattern 5: Reading Context in the Same Component as Provider

**Problem**: `useContext` searches UPWARD from the calling component. The Provider in the same component does NOT affect that component's own `useContext` call.

```tsx
// BAD -- useTheme reads from PARENT provider, not this component's provider
function App(): JSX.Element {
  const theme = useTheme(); // reads from ABOVE, not below

  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}
```

**Fix**: Move the consumer into a child component:

```tsx
// GOOD -- consumer is below the provider
function App(): JSX.Element {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedApp />
    </ThemeContext.Provider>
  );
}

function ThemedApp(): JSX.Element {
  const { theme } = useTheme(); // correctly reads "dark"
  return <Page className={theme} />;
}
```

---

## Anti-Pattern 6: Using Context.Consumer

**Problem**: The render prop pattern (`Context.Consumer`) is legacy and verbose.

```tsx
// BAD -- legacy pattern, verbose, hard to compose
function Header(): JSX.Element {
  return (
    <ThemeContext.Consumer>
      {(theme) => (
        <AuthContext.Consumer>
          {(auth) => (
            <header className={theme}>
              Welcome, {auth.user?.name}
            </header>
          )}
        </AuthContext.Consumer>
      )}
    </ThemeContext.Consumer>
  );
}
```

**Fix**: ALWAYS use `useContext` hook:

```tsx
// GOOD -- clean, composable
function Header(): JSX.Element {
  const { theme } = useTheme();
  const { user } = useAuth();

  return (
    <header className={theme}>
      Welcome, {user?.name}
    </header>
  );
}
```

---

## Anti-Pattern 7: Prop Drilling Through Context

**Problem**: Using Context to pass data that only goes one or two levels deep. Context adds complexity without benefit for shallow trees.

```tsx
// BAD -- unnecessary context for shallow tree
const ButtonColorContext = createContext<string>("blue");

function Card(): JSX.Element {
  return (
    <ButtonColorContext.Provider value="red">
      <CardBody />
    </ButtonColorContext.Provider>
  );
}

function CardBody(): JSX.Element {
  const color = useContext(ButtonColorContext);
  return <Button color={color} />;
}
```

**Fix**: Pass props directly for shallow hierarchies:

```tsx
// GOOD -- simple prop passing for 1-2 levels
function Card(): JSX.Element {
  return <CardBody buttonColor="red" />;
}

function CardBody({ buttonColor }: { buttonColor: string }): JSX.Element {
  return <Button color={buttonColor} />;
}
```

**Rule of thumb**: If the data passes through fewer than 3 levels, prefer props over Context.

---

## Anti-Pattern 8: Duplicate Module Instances

**Problem**: If the module containing `createContext` is bundled twice (e.g., different versions in node_modules), the Provider and Consumer reference DIFFERENT context objects.

**Symptoms**:
- `useContext` returns the default value despite Provider being present
- No error messages -- silently fails

**Fix**:
- Ensure single module instance in your bundler configuration
- Use `npm ls react` to check for duplicate React installations
- In monorepos, hoist shared dependencies to the root

---

## Anti-Pattern 9: Missing value Prop on Provider

**Problem**: Rendering a Provider without a `value` prop passes `undefined` to consumers, overriding the `defaultValue` from `createContext`.

```tsx
// BAD -- passes undefined, NOT the defaultValue from createContext
<ThemeContext.Provider>
  <App />
</ThemeContext.Provider>
```

**Why**: `defaultValue` from `createContext` is ONLY used when there is NO Provider in the tree at all. A Provider without `value` explicitly passes `undefined`.

**Fix**: ALWAYS pass an explicit `value` prop:

```tsx
// GOOD
<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>
```

---

## Summary Table

| Anti-Pattern | Impact | Severity |
|-------------|--------|----------|
| Unstable value object | All consumers re-render every parent render | High |
| Missing provider guard | Silent null bugs, confusing runtime errors | High |
| Single giant context | Unrelated components re-render together | Medium |
| Frequently changing values | Performance degradation, janky UI | High |
| Same-component provider/consumer | Reads wrong context value | Medium |
| Using Context.Consumer | Verbose code, callback hell | Low |
| Context for shallow props | Unnecessary complexity | Low |
| Duplicate module instances | Silent context mismatch | High |
| Missing value prop | Consumers receive undefined | Medium |
