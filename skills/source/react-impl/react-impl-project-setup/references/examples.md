# Project Setup Workflows and Configuration Examples

## Example 1: Minimal React 18 + Vite + TypeScript Setup

```bash
# Step 1: Scaffold the project
npm create vite@latest my-app -- --template react-ts

# Step 2: Install dependencies
cd my-app
npm install

# Step 3: Start development server
npm run dev
```

Generated project includes:
- `src/App.tsx` -- root component
- `src/main.tsx` -- entry point with `createRoot`
- `index.html` -- Vite HTML entry
- `vite.config.ts` -- Vite config with `@vitejs/plugin-react`
- `tsconfig.json` -- project references root
- `tsconfig.app.json` -- app TypeScript config
- `tsconfig.node.json` -- Node/Vite TypeScript config

---

## Example 2: React 19 Project Setup

```bash
# Step 1: Scaffold with Vite
npm create vite@latest my-app-19 -- --template react-ts
cd my-app-19

# Step 2: Upgrade to React 19
npm install react@19 react-dom@19

# Step 3: Install React 19 types (transition period)
npm install -D @types/react@19 @types/node

# Step 4: Verify version
npm run dev
```

```typescript
// src/main.tsx -- React 19 entry point (same API as React 18)
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

---

## Example 3: Path Aliases Configuration

### Step 1: TypeScript (tsconfig.app.json)

```jsonc
{
  "compilerOptions": {
    // ... other options
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Step 2: Vite (vite.config.ts)

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### Step 3: Usage

```typescript
// Before (fragile relative imports)
import { Button } from '../../../components/ui/Button';
import { useAuth } from '../../hooks/useAuth';

// After (clean alias imports)
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
```

---

## Example 4: Environment Variables Setup

### .env

```bash
# Committed to git -- no secrets here
VITE_APP_TITLE=My React App
VITE_API_URL=https://api.example.com
```

### .env.local

```bash
# NEVER committed -- local development overrides
VITE_API_URL=http://localhost:8080
SECRET_KEY=abc123  # NOT exposed to client (no VITE_ prefix)
```

### .env.production

```bash
# Production-specific values
VITE_API_URL=https://api.production.com
VITE_APP_TITLE=My React App (Production)
```

### Type Declarations (src/env.d.ts)

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string;
  readonly VITE_API_URL: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Usage in Components

```tsx
function ApiStatus(): React.ReactElement {
  const apiUrl = import.meta.env.VITE_API_URL;
  const isDev = import.meta.env.DEV;

  return (
    <div>
      <p>API: {apiUrl}</p>
      {isDev && <p>Running in development mode</p>}
    </div>
  );
}
```

---

## Example 5: Complete ESLint Flat Config

```bash
# Install all ESLint dependencies
npm install -D eslint @eslint/js typescript-eslint \
  eslint-plugin-react-hooks eslint-plugin-react-refresh
```

```typescript
// eslint.config.js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';

export default tseslint.config(
  { ignores: ['dist'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
      // Additional recommended rules
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_', varsIgnorePattern: '^_' },
      ],
      '@typescript-eslint/consistent-type-imports': [
        'error',
        { prefer: 'type-imports' },
      ],
    },
  },
);
```

---

## Example 6: Vite Dev Server with API Proxy

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    open: true,
    proxy: {
      // Proxy /api requests to backend
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      // Proxy WebSocket connections
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
  build: {
    sourcemap: true,
    target: 'es2020',
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
        },
      },
    },
  },
});
```

---

## Example 7: Complete package.json

```jsonc
{
  "name": "my-react-app",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "type-check": "tsc -b --noEmit",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,json}\""
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@eslint/js": "^9.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@types/node": "^22.0.0",
    "@vitejs/plugin-react": "^4.3.0",
    "eslint": "^9.0.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.0",
    "prettier": "^3.3.0",
    "typescript": "~5.6.0",
    "typescript-eslint": "^8.0.0",
    "vite": "^6.0.0"
  }
}
```

---

## Example 8: Barrel Exports Pattern

```typescript
// src/components/ui/Button.tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  children: React.ReactNode;
  onClick?: () => void;
}

export function Button({ variant = 'primary', children, onClick }: ButtonProps): React.ReactElement {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

```typescript
// src/components/ui/index.ts -- barrel export
export { Button } from './Button';
export { Input } from './Input';
export { Card } from './Card';
```

```typescript
// src/components/index.ts -- top-level barrel
export * from './ui';
export * from './features';
```

```typescript
// Usage in a page component
import { Button, Input, Card } from '@/components';
```

---

## Example 9: Prettier Configuration

```bash
npm install -D prettier
```

```jsonc
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always"
}
```

```
// .prettierignore
dist
node_modules
*.min.js
```
