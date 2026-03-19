---
name: react-impl-project-setup
description: >
  Use when creating a new React project, configuring Vite, setting up TypeScript,
  or establishing project conventions. Prevents the common mistake of inconsistent
  tooling setup or missing essential configuration like path aliases and linting.
  Covers Vite setup, TypeScript config, ESLint, Prettier, path aliases,
  environment variables, folder structure conventions.
  Keywords: Vite, TypeScript, ESLint, Prettier, project setup, path aliases.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript and Vite."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-project-setup

## Quick Reference

### Project Creation (Step-by-Step)

#### React 18 (Stable)

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

#### React 19

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install react@19 react-dom@19
npm install -D @types/react@19 @types/node
npm run dev
```

> In React 19, type definitions are included in the `react` package itself. Install `@types/react@19` for the transition period until all tooling catches up.

### Critical Warnings

**NEVER** use Create React App (CRA) -- it is deprecated and unmaintained. ALWAYS use Vite with `react-ts` template for new React projects.

**NEVER** commit `.env.local` or `.env.*.local` files -- these contain local secrets. ALWAYS add them to `.gitignore`.

**NEVER** expose sensitive keys in environment variables without the `VITE_` prefix check -- only variables prefixed with `VITE_` are exposed to the client bundle.

**NEVER** skip `strict: true` in `tsconfig.json` -- strict mode catches entire categories of bugs at compile time. ALWAYS enable it for new projects.

**NEVER** use relative imports like `../../../components/Button` -- ALWAYS configure path aliases (`@/`) to keep imports clean and refactor-safe.

**ALWAYS** add `type-check` as a separate script in `package.json` -- Vite does NOT perform type checking during development or build by default.

---

## Recommended Project Structure

```
my-app/
├── public/                     # Static assets (copied as-is to dist/)
│   └── favicon.svg
├── src/
│   ├── components/             # Reusable UI components
│   │   ├── ui/                 # Primitive/base components (Button, Input, Card)
│   │   └── features/           # Feature-specific components
│   ├── hooks/                  # Custom React hooks
│   ├── utils/                  # Pure utility functions (no React dependency)
│   ├── types/                  # Shared TypeScript type definitions
│   ├── pages/                  # Route-level page components
│   ├── layouts/                # Layout wrapper components
│   ├── services/               # API calls and external service integrations
│   ├── stores/                 # State management (context, zustand, etc.)
│   ├── constants/              # Application constants and config
│   ├── assets/                 # Images, fonts, SVGs imported in code
│   ├── App.tsx                 # Root component
│   ├── main.tsx                # Application entry point
│   └── vite-env.d.ts           # Vite client type declarations
├── .env                        # Default env vars (committed, no secrets)
├── .env.local                  # Local overrides (NEVER commit)
├── .env.production             # Production env vars
├── .gitignore
├── .eslintrc.cjs               # ESLint configuration
├── index.html                  # HTML entry point (Vite uses this as root)
├── package.json
├── tsconfig.json               # TypeScript config (project references root)
├── tsconfig.app.json           # App-specific TS config
├── tsconfig.node.json          # Node/Vite config TS settings
└── vite.config.ts              # Vite configuration
```

### Structure Rules

- **ALWAYS** co-locate component tests next to their source: `Button.tsx` + `Button.test.tsx`
- **ALWAYS** use barrel exports (`index.ts`) per directory for clean imports
- **ALWAYS** keep `utils/` free of React imports -- pure functions only
- **ALWAYS** place route-level components in `pages/`, reusable components in `components/`

---

## TypeScript Configuration

### tsconfig.json (Project References Root)

```jsonc
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

### tsconfig.app.json

```jsonc
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,

    /* Path aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

### Key Settings Explained

| Setting | Value | Why |
|---------|-------|-----|
| `strict` | `true` | Enables all strict type-checking options |
| `jsx` | `"react-jsx"` | Uses React 17+ automatic JSX transform (no `import React` needed) |
| `moduleResolution` | `"bundler"` | Matches Vite's module resolution strategy |
| `isolatedModules` | `true` | Required for Vite -- ensures files can be transpiled independently |
| `noEmit` | `true` | Vite handles transpilation; TypeScript only type-checks |
| `noUncheckedIndexedAccess` | `true` | Prevents unsafe array/object index access |
| `paths` | `@/* -> ./src/*` | Path alias for clean imports |

---

## Vite Configuration

### vite.config.ts

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
  server: {
    port: 3000,
    // Proxy API requests during development
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
  build: {
    sourcemap: true,
    target: 'es2020',
  },
});
```

### Path Alias Sync

**ALWAYS** configure path aliases in BOTH `tsconfig.app.json` AND `vite.config.ts` -- TypeScript needs them for type checking, Vite needs them for bundling.

---

## ESLint Configuration

### Install Dependencies

```bash
npm install -D eslint @eslint/js typescript-eslint \
  eslint-plugin-react-hooks eslint-plugin-react-refresh
```

### eslint.config.js (Flat Config)

```typescript
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
    },
  },
);
```

> **ALWAYS** use ESLint flat config (`eslint.config.js`) for new projects -- the `.eslintrc.*` format is deprecated.

---

## Environment Variables

### Rules

| Rule | Detail |
|------|--------|
| Client-exposed vars | MUST be prefixed with `VITE_` |
| Access pattern | `import.meta.env.VITE_API_URL` |
| Server-only vars | Use vars WITHOUT `VITE_` prefix -- they are NOT bundled |
| Built-in vars | `import.meta.env.MODE`, `import.meta.env.DEV`, `import.meta.env.PROD` |

### .env File Load Order

1. `.env` -- loaded in all cases
2. `.env.local` -- loaded in all cases, ignored by git
3. `.env.[mode]` -- loaded for specified mode (development/production)
4. `.env.[mode].local` -- loaded for specified mode, ignored by git

### Type Declarations for Env Vars

```typescript
// src/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

**ALWAYS** declare custom `VITE_` variables in a type definition file -- this provides autocompletion and type safety for `import.meta.env`.

---

## Package.json Scripts

```jsonc
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "type-check": "tsc -b --noEmit"
  }
}
```

| Script | Purpose |
|--------|---------|
| `dev` | Start Vite dev server with HMR |
| `build` | Type-check then build for production |
| `preview` | Locally preview production build |
| `lint` | Run ESLint across the project |
| `type-check` | Run TypeScript compiler for type checking only |

---

## .gitignore

```gitignore
# Dependencies
node_modules/

# Build output
dist/
dist-ssr/

# Environment (local overrides with secrets)
.env.local
.env.*.local

# Editor
.vscode/*
!.vscode/extensions.json
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Debug logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

---

## Decision Trees

### New Project Setup

```
Need a new React project?
├── ALWAYS use: npm create vite@latest -- --template react-ts
├── Need React 19?
│   └── After scaffold: npm install react@19 react-dom@19
├── Need path aliases?
│   └── Configure BOTH tsconfig.app.json AND vite.config.ts
├── Need API proxy?
│   └── Add server.proxy to vite.config.ts
└── Need env variables in client code?
    └── Prefix with VITE_ and declare types in env.d.ts
```

### TypeScript Strictness

```
Setting up tsconfig?
├── ALWAYS enable strict: true
├── ALWAYS enable isolatedModules: true (Vite requirement)
├── ALWAYS set jsx: "react-jsx" (automatic transform)
├── ALWAYS set noEmit: true (Vite handles transpilation)
└── Consider noUncheckedIndexedAccess: true (safer array access)
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete project setup workflows and configuration examples
- [references/patterns.md](references/patterns.md) -- Project structure patterns and conventions
- [references/anti-patterns.md](references/anti-patterns.md) -- Common setup mistakes and why they fail

### Official Sources

- https://vite.dev/guide/
- https://react.dev/learn/start-a-new-react-project
- https://react.dev/learn/typescript
- https://www.typescriptlang.org/tsconfig
- https://eslint.org/docs/latest/use/configure/configuration-files
