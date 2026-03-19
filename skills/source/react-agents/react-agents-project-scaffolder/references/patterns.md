# Project Structure Patterns

Rationale and guidelines for React project organization decisions.

---

## Pattern 1: Feature-Based Organization (Medium/Large)

### Structure

```
src/features/
├── auth/
│   ├── components/
│   │   ├── LoginForm.tsx
│   │   └── SignupForm.tsx
│   ├── hooks/
│   │   └── useAuth.ts
│   ├── api.ts
│   ├── store.ts
│   ├── types.ts
│   └── index.ts          # Public API barrel export
├── dashboard/
│   ├── components/
│   ├── hooks/
│   ├── api.ts
│   ├── store.ts
│   ├── types.ts
│   └── index.ts
```

### Rules

- **ALWAYS** co-locate feature-specific components, hooks, API calls, and types within the feature directory
- **ALWAYS** use an `index.ts` barrel file to define the public API of each feature
- **NEVER** import internal feature files from outside the feature -- ALWAYS import from `@/features/{name}`
- **NEVER** create cross-feature dependencies without going through the public API

### Rationale

Feature-based organization scales because:
- Adding a new feature means adding a new directory, not modifying shared directories
- Deleting a feature is a single directory removal
- Each feature is self-contained and independently testable
- Code review scope is clear per feature

---

## Pattern 2: Flat Organization (Small)

### Structure

```
src/
├── components/
│   ├── App.tsx
│   ├── Header.tsx
│   └── TodoList.tsx
├── hooks/
│   └── useTodos.ts
├── types/
│   └── index.ts
└── utils/
    └── format.ts
```

### Rules

- **ALWAYS** use flat organization for projects with fewer than 20 components
- **NEVER** create feature directories for a small project -- it adds unnecessary nesting
- **ALWAYS** migrate to feature-based when component count exceeds 20 or team exceeds 3

### Rationale

Flat structure works for small projects because:
- Every file is reachable within two levels of nesting
- No barrel files to maintain
- Lower cognitive overhead for solo developers

---

## Pattern 3: Shared UI Components

### Structure

```
src/components/ui/
├── Button/
│   ├── Button.tsx
│   ├── Button.module.css
│   └── Button.test.tsx
├── Input/
│   ├── Input.tsx
│   ├── Input.module.css
│   └── Input.test.tsx
├── Modal/
│   ├── Modal.tsx
│   ├── Modal.module.css
│   └── Modal.test.tsx
└── index.ts               # Re-exports all UI components
```

### Rules

- **ALWAYS** co-locate component, styles, and tests in the same directory
- **ALWAYS** name the directory and component file identically (PascalCase)
- **ALWAYS** export shared UI components from `src/components/ui/index.ts`
- **NEVER** put business logic in shared UI components -- they MUST be purely presentational
- **NEVER** import feature-specific types into shared UI components

### Rationale

Component co-location means:
- Related files are always together (no hunting across directories)
- Deleting a component removes all its artifacts
- Test files are discoverable without separate test directory mirroring

---

## Pattern 4: Path Aliases

### Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

```typescript
// vite.config.ts
import { resolve } from "path";

export default defineConfig({
  resolve: {
    alias: {
      "@": resolve(__dirname, "src"),
    },
  },
});
```

### Rules

- **ALWAYS** configure the `@` alias pointing to `src/`
- **ALWAYS** configure both `tsconfig.json` and `vite.config.ts` -- they are independent
- **ALWAYS** use `@/` for imports that cross directory boundaries
- **NEVER** use relative paths with more than one `../` level -- use `@/` instead
- **NEVER** create additional aliases unless the project has multiple source roots

### Import Examples

```typescript
// GOOD -- clear origin
import { Button } from "@/components/ui";
import { useAuth } from "@/features/auth";
import { apiClient } from "@/lib/api-client";

// GOOD -- relative for sibling files
import { LoginForm } from "./LoginForm";
import styles from "./Login.module.css";

// BAD -- deep relative paths
import { Button } from "../../../components/ui/Button";
```

---

## Pattern 5: Test Organization

### Structure

```
project-root/
├── src/
│   └── components/ui/Button/
│       ├── Button.tsx
│       └── Button.test.tsx        # Unit tests co-located
├── test/
│   ├── setup.ts                   # Vitest setup file
│   └── test-utils.tsx             # Custom render with providers
└── e2e/                           # Large projects only
    ├── auth.spec.ts
    └── playwright.config.ts
```

### Rules

- **ALWAYS** co-locate unit tests next to the source file (`Component.test.tsx`)
- **ALWAYS** place test setup and utilities in the top-level `test/` directory
- **NEVER** place `setup.ts` inside `src/` -- it pollutes the application source
- **ALWAYS** create a `test-utils.tsx` that wraps render with all providers (QueryClient, Router, etc.)
- **ALWAYS** use the custom `renderWithProviders` in tests instead of bare `render`
- **NEVER** mock React hooks directly -- test behavior through the component

### Vitest Configuration

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    environment: "jsdom",
    setupFiles: ["./test/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
    css: true,
  },
});
```

---

## Pattern 6: Environment Variables

### Rules

- **ALWAYS** prefix client-side environment variables with `VITE_` -- Vite only exposes prefixed variables
- **ALWAYS** create `.env.example` with placeholder values and commit it
- **NEVER** commit `.env` or `.env.local` -- they MUST be in `.gitignore`
- **ALWAYS** access variables via `import.meta.env.VITE_*`
- **ALWAYS** validate required environment variables at startup

### Validation Pattern

```typescript
// src/lib/env.ts
function getEnvVar(key: string): string {
  const value = import.meta.env[key];
  if (!value) {
    throw new Error(`Missing environment variable: ${key}`);
  }
  return value;
}

export const env = {
  apiUrl: getEnvVar("VITE_API_URL"),
} as const;
```

---

## Pattern 7: Barrel Exports

### Rules

- **ALWAYS** use barrel exports (`index.ts`) for feature and UI component directories
- **NEVER** re-export internal implementation details -- only the public API
- **NEVER** use barrel exports in leaf directories with fewer than 3 files

### Example

```typescript
// src/features/auth/index.ts
export { LoginForm } from "./components/LoginForm";
export { SignupForm } from "./components/SignupForm";
export { useAuth } from "./hooks/useAuth";
export type { User, AuthState } from "./types";
// Do NOT export: ./api.ts internals, ./store.ts internals
```

---

## Pattern 8: State Management Layers

### Decision Matrix

| State Type | Location | Tool |
|------------|----------|------|
| Component UI state (open/close, form inputs) | Component | `useState` |
| Complex component state (reducers) | Component | `useReducer` |
| Shared client state (theme, sidebar, user prefs) | Global store | Zustand |
| Server/async state (API data, caching) | Query layer | TanStack Query |
| URL state (filters, pagination, search) | URL | React Router `useSearchParams` |

### Rules

- **NEVER** put server state in Zustand -- ALWAYS use TanStack Query for API data
- **NEVER** put UI state in global stores -- ALWAYS keep it local to the component
- **ALWAYS** separate client state (Zustand) from server state (TanStack Query)
- **NEVER** duplicate URL-derivable state in a store -- read from `useSearchParams`

---

## Pattern 9: Providers Composition

### Rules

- **ALWAYS** compose providers in a single `Providers` component (`src/app/providers.tsx`)
- **ALWAYS** order providers from outermost (most general) to innermost (most specific)
- **NEVER** nest providers inside feature components -- they belong at the app level

### Recommended Order

```tsx
export function Providers({ children }: { children: ReactNode }) {
  return (
    <ErrorBoundary>           {/* 1. Error catching (outermost) */}
      <QueryClientProvider>   {/* 2. Data layer */}
        <AuthProvider>        {/* 3. Authentication */}
          <ThemeProvider>     {/* 4. UI theming */}
            {children}
          </ThemeProvider>
        </AuthProvider>
      </QueryClientProvider>
    </ErrorBoundary>
  );
}
```

---

## Pattern 10: Routing Architecture

### Rules

- **ALWAYS** use `createBrowserRouter` from React Router v6 -- it enables data loading and error boundaries per route
- **ALWAYS** define routes in a single `router.tsx` file for small/medium projects
- **NEVER** use the legacy `<BrowserRouter>` + `<Routes>` pattern for new projects
- **ALWAYS** include a catch-all 404 route as the last child
- **ALWAYS** use layout routes with `<Outlet />` to share navigation and layout

### Route Organization (Large Projects)

```typescript
// src/app/router.tsx
import { createBrowserRouter } from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    lazy: () => import("@/app/layouts/RootLayout"),
    children: [
      { index: true, lazy: () => import("@/features/home") },
      {
        path: "dashboard",
        lazy: () => import("@/features/dashboard"),
        children: [
          { index: true, lazy: () => import("@/features/dashboard/Overview") },
          { path: "settings", lazy: () => import("@/features/dashboard/Settings") },
        ],
      },
      { path: "*", lazy: () => import("@/app/pages/NotFound") },
    ],
  },
]);
```

- **ALWAYS** use `lazy()` for route-level code splitting in medium/large projects
- **NEVER** eagerly import all route components -- it defeats code splitting
