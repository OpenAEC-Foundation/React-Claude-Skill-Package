# Project Structure Patterns

## Pattern 1: Feature-Based Organization

Organize code by feature/domain rather than by file type. Each feature directory contains its own components, hooks, types, and utilities.

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── LoginForm.test.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── types.ts
│   │   ├── api.ts
│   │   └── index.ts
│   ├── dashboard/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── types.ts
│   │   └── index.ts
│   └── settings/
│       ├── components/
│       ├── hooks/
│       └── index.ts
├── components/              # Shared/generic components only
│   └── ui/
├── hooks/                   # Shared hooks only
├── utils/                   # Shared utilities only
└── types/                   # Global types only
```

### When to Use

- **ALWAYS** use feature-based organization for projects with 3+ distinct features
- **ALWAYS** keep shared/generic code in top-level directories (`components/`, `hooks/`, `utils/`)
- **NEVER** import from another feature's internal files -- use only the barrel export (`index.ts`)

### Benefits

- Features are self-contained and can be moved, deleted, or extracted
- Reduces cross-feature coupling
- Makes code ownership and review boundaries clear

---

## Pattern 2: Flat Component Organization (Small Projects)

For small projects (< 15 components), a flat structure is simpler and avoids premature abstraction.

```
src/
├── components/
│   ├── Button.tsx
│   ├── Button.test.tsx
│   ├── Header.tsx
│   ├── Footer.tsx
│   ├── TodoList.tsx
│   └── TodoItem.tsx
├── hooks/
│   └── useTodos.ts
├── utils/
│   └── formatDate.ts
├── App.tsx
└── main.tsx
```

### When to Use

- Projects with fewer than 15 components
- Prototypes and MVPs
- **ALWAYS** migrate to feature-based when the flat structure exceeds ~20 files in `components/`

---

## Pattern 3: Component Co-location

ALWAYS keep related files next to their component. Tests, styles, and stories belong next to the component, not in separate top-level directories.

```
src/components/ui/Button/
├── Button.tsx              # Component implementation
├── Button.test.tsx         # Unit tests
├── Button.module.css       # Scoped styles (if using CSS Modules)
├── Button.stories.tsx      # Storybook stories (if using Storybook)
└── index.ts                # Re-export
```

```typescript
// Button/index.ts
export { Button } from './Button';
export type { ButtonProps } from './Button';
```

### Rules

- **ALWAYS** co-locate test files: `Component.test.tsx` next to `Component.tsx`
- **ALWAYS** co-locate styles: `Component.module.css` next to `Component.tsx`
- **NEVER** create a separate `__tests__/` or `styles/` directory at the project root
- **ALWAYS** provide an `index.ts` barrel export for component directories

---

## Pattern 4: Layout Pattern

Layouts wrap page content with shared UI elements (navigation, sidebar, footer). Use layouts to avoid repeating wrapper markup in every page.

```
src/
├── layouts/
│   ├── RootLayout.tsx        # App shell: nav + outlet
│   ├── DashboardLayout.tsx   # Sidebar + content area
│   └── AuthLayout.tsx        # Centered card for login/signup
├── pages/
│   ├── HomePage.tsx
│   ├── DashboardPage.tsx
│   └── LoginPage.tsx
```

```tsx
// layouts/RootLayout.tsx
import { Outlet } from 'react-router-dom';
import { Header } from '@/components/Header';
import { Footer } from '@/components/Footer';

export function RootLayout(): React.ReactElement {
  return (
    <div className="app">
      <Header />
      <main>
        <Outlet />
      </main>
      <Footer />
    </div>
  );
}
```

### Rules

- **ALWAYS** separate layouts from pages -- layouts handle structure, pages handle content
- **ALWAYS** use `<Outlet />` (React Router) or `{children}` for content injection
- **NEVER** put data fetching logic in layout components -- layouts are structural only

---

## Pattern 5: Service Layer Pattern

Centralize API calls and external service interactions in a `services/` directory. Components and hooks consume services, never call `fetch` directly.

```typescript
// src/services/api.ts -- base API configuration
const BASE_URL = import.meta.env.VITE_API_URL;

async function request<T>(endpoint: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${BASE_URL}${endpoint}`, {
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    ...options,
  });

  if (!response.ok) {
    throw new Error(`API error: ${response.status} ${response.statusText}`);
  }

  return response.json() as Promise<T>;
}

export const api = { request };
```

```typescript
// src/services/todos.ts -- domain-specific service
import { api } from './api';
import type { Todo, CreateTodoInput } from '@/types';

export const todoService = {
  getAll: () => api.request<Todo[]>('/todos'),
  getById: (id: string) => api.request<Todo>(`/todos/${id}`),
  create: (input: CreateTodoInput) =>
    api.request<Todo>('/todos', {
      method: 'POST',
      body: JSON.stringify(input),
    }),
  delete: (id: string) =>
    api.request<void>(`/todos/${id}`, { method: 'DELETE' }),
};
```

### Rules

- **ALWAYS** centralize API base URL and default headers in one file
- **ALWAYS** type all request and response payloads
- **NEVER** call `fetch()` directly in components -- use the service layer
- **NEVER** store auth tokens in service files -- inject them via interceptors or headers

---

## Pattern 6: Type Organization

```
src/types/
├── index.ts          # Re-exports all shared types
├── api.ts            # API response/request types
├── models.ts         # Domain model types
└── common.ts         # Utility types (Nullable, Optional, etc.)
```

```typescript
// src/types/models.ts
export interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user' | 'viewer';
  createdAt: string;
}

export interface Todo {
  id: string;
  title: string;
  completed: boolean;
  userId: string;
}
```

```typescript
// src/types/api.ts
export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}

export interface ApiError {
  message: string;
  code: string;
  status: number;
}
```

### Rules

- **ALWAYS** use `interface` for object shapes, `type` for unions and intersections
- **ALWAYS** export types from barrel `index.ts`
- **NEVER** put React component prop types in `types/` -- keep prop types co-located with their component
- **ALWAYS** use `type` imports: `import type { User } from '@/types'`

---

## Pattern 7: Constants and Configuration

```typescript
// src/constants/index.ts
export const APP_CONFIG = {
  APP_NAME: 'My React App',
  MAX_UPLOAD_SIZE_MB: 10,
  PAGINATION_DEFAULT_SIZE: 20,
  DEBOUNCE_MS: 300,
} as const;

export const ROUTES = {
  HOME: '/',
  LOGIN: '/login',
  DASHBOARD: '/dashboard',
  SETTINGS: '/settings',
} as const;

export const QUERY_KEYS = {
  TODOS: 'todos',
  USERS: 'users',
  USER_PROFILE: 'user-profile',
} as const;
```

### Rules

- **ALWAYS** use `as const` for constant objects -- ensures literal types and readonly
- **ALWAYS** centralize route paths in a constants file
- **NEVER** hardcode magic numbers or strings in components -- extract to constants
