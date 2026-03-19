---
name: react-agents-project-scaffolder
description: >
  Use when scaffolding a new React project, adding React to an existing project,
  or generating a feature-complete application structure. Prevents the common
  mistake of inconsistent project setup with missing TypeScript, linting, or test
  configuration. Covers Vite config, TypeScript setup, component architecture,
  routing, state management, testing infrastructure, folder structure.
  Keywords: scaffold, Vite, project structure, TypeScript config, folder layout.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript and Vite."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-agents-project-scaffolder

## Quick Reference

### Project Size Classification

| Size | Routes | Components | State Complexity | Team |
|------|--------|------------|-----------------|------|
| Small | 1-5 | < 20 | Local state only | 1-2 devs |
| Medium | 5-20 | 20-80 | Shared + server state | 3-8 devs |
| Large | 20+ | 80+ | Complex orchestration | 8+ devs |

### Generated Stack Summary

| Layer | Small | Medium | Large |
|-------|-------|--------|-------|
| Build | Vite + @vitejs/plugin-react | Vite + @vitejs/plugin-react | Vite + @vitejs/plugin-react |
| Language | TypeScript (strict) | TypeScript (strict) | TypeScript (strict) |
| Routing | React Router v6 | React Router v6 | React Router v6 |
| Client State | useState/useReducer | Zustand | Zustand |
| Server State | fetch + useEffect | TanStack Query | TanStack Query |
| Styling | CSS Modules | CSS Modules | CSS Modules |
| Testing | Vitest + RTL | Vitest + RTL | Vitest + RTL + Playwright |
| Linting | ESLint flat config | ESLint flat config | ESLint flat config |

### Critical Warnings

**NEVER** scaffold with Create React App -- it is deprecated and unmaintained. ALWAYS use Vite with `@vitejs/plugin-react`.

**NEVER** use default `tsconfig.json` -- ALWAYS enable `strict: true`, `noUncheckedIndexedAccess: true`, and configure path aliases.

**NEVER** install both `@types/react` and React 19 -- React 19 ships built-in TypeScript types. ALWAYS check the React version before adding `@types/react`.

**NEVER** mix CSS-in-JS runtime libraries (styled-components, emotion) with React Server Components -- they require client-side JavaScript. ALWAYS use CSS Modules or Tailwind for RSC-compatible projects.

**NEVER** place test setup files inside `src/` -- ALWAYS place `setup.ts` at the project root or in a dedicated `test/` directory.

---

## Project Size Decision Tree

```
START: What is the project scope?
│
├─ Prototype / landing page / single feature?
│  └─ → SMALL project scaffold
│
├─ Multi-page app with auth, forms, API integration?
│  └─ → MEDIUM project scaffold
│
├─ Enterprise app with complex state, many teams, SSR needs?
│  └─ → LARGE project scaffold
│
└─ Unsure?
   └─ → Default to MEDIUM (scales both directions)
```

### SSR Decision

```
Does the project need SSR or static generation?
│
├─ YES → Use a React framework (Next.js, Remix, TanStack Start)
│        This skill generates the SPA scaffold only.
│        Adapt the patterns below to your framework's conventions.
│
└─ NO  → Continue with Vite SPA scaffold below.
```

---

## Small Project Structure

```
project-root/
├── public/
│   └── favicon.svg
├── src/
│   ├── components/
│   │   ├── App.tsx
│   │   ├── App.module.css
│   │   └── {Component}.tsx
│   ├── hooks/
│   │   └── use{Hook}.ts
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   └── {util}.ts
│   ├── main.tsx
│   └── index.css
├── test/
│   └── setup.ts
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
├── eslint.config.js
├── .prettierrc
├── .gitignore
├── .env.example
└── package.json
```

---

## Medium Project Structure

```
project-root/
├── public/
│   └── favicon.svg
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── router.tsx
│   │   └── providers.tsx
│   ├── components/
│   │   └── ui/
│   │       ├── Button/
│   │       │   ├── Button.tsx
│   │       │   ├── Button.module.css
│   │       │   └── Button.test.tsx
│   │       └── index.ts
│   ├── features/
│   │   └── {feature}/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api.ts
│   │       ├── store.ts
│   │       ├── types.ts
│   │       └── index.ts
│   ├── hooks/
│   │   └── use{Hook}.ts
│   ├── lib/
│   │   ├── api-client.ts
│   │   └── query-client.ts
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   └── {util}.ts
│   ├── styles/
│   │   ├── tokens.css
│   │   └── global.css
│   ├── main.tsx
│   └── vite-env.d.ts
├── test/
│   ├── setup.ts
│   └── test-utils.tsx
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
├── eslint.config.js
├── .prettierrc
├── .gitignore
├── .env.example
└── package.json
```

---

## Large Project Structure

```
project-root/
├── public/
│   └── favicon.svg
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── router.tsx
│   │   ├── providers.tsx
│   │   └── error-boundary.tsx
│   ├── components/
│   │   └── ui/
│   │       ├── Button/
│   │       ├── Input/
│   │       ├── Modal/
│   │       └── index.ts
│   ├── features/
│   │   └── {feature}/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api.ts
│   │       ├── store.ts
│   │       ├── types.ts
│   │       └── index.ts
│   ├── hooks/
│   ├── lib/
│   │   ├── api-client.ts
│   │   ├── query-client.ts
│   │   └── auth.ts
│   ├── types/
│   │   ├── api.ts
│   │   └── index.ts
│   ├── utils/
│   ├── styles/
│   │   ├── tokens.css
│   │   └── global.css
│   ├── main.tsx
│   └── vite-env.d.ts
├── test/
│   ├── setup.ts
│   ├── test-utils.tsx
│   └── mocks/
│       └── handlers.ts
├── e2e/
│   ├── {feature}.spec.ts
│   └── playwright.config.ts
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
├── eslint.config.js
├── .prettierrc
├── .gitignore
├── .env.example
└── package.json
```

---

## Core Configuration Templates

### vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { resolve } from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": resolve(__dirname, "src"),
    },
  },
  server: {
    port: 3000,
    strictPort: true,
  },
  build: {
    sourcemap: true,
    target: "es2022",
  },
});
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src", "test"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### tsconfig.node.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "strict": true,
    "composite": true
  },
  "include": ["vite.config.ts", "eslint.config.js"]
}
```

---

## Dependency Decision Tree

### React 18.x Dependencies

```
ALWAYS install:
├── react@^18.3.0
├── react-dom@^18.3.0
├── @types/react@^18.3.0         (required -- React 18 has no built-in types)
├── @types/react-dom@^18.3.0
├── typescript@^5.5.0
├── vite@^6.0.0
├── @vitejs/plugin-react@^4.0.0
├── eslint@^9.0.0
├── prettier@^3.0.0
└── vitest@^2.0.0

If MEDIUM or LARGE, also install:
├── react-router-dom@^6.28.0
├── @tanstack/react-query@^5.0.0
├── @testing-library/react@^16.0.0
├── @testing-library/jest-dom@^6.0.0
├── @testing-library/user-event@^14.0.0
└── jsdom@^25.0.0

If MEDIUM or LARGE with shared client state:
└── zustand@^5.0.0

If LARGE, also install:
├── @playwright/test@^1.48.0
└── msw@^2.0.0
```

### React 19.x Dependencies

```
ALWAYS install:
├── react@^19.0.0
├── react-dom@^19.0.0
├── (NO @types/react -- React 19 ships built-in types)
├── (NO @types/react-dom -- React 19 ships built-in types)
├── typescript@^5.5.0
├── vite@^6.0.0
├── @vitejs/plugin-react@^4.0.0
├── eslint@^9.0.0
├── prettier@^3.0.0
└── vitest@^2.0.0

Remaining dependencies are the same as React 18.x above.
```

**ALWAYS** check the target React version before generating `package.json`. The `@types/react` difference between React 18 and 19 causes type conflicts if installed incorrectly.

---

## Styling Decision

```
Default: CSS Modules
│
├─ Team already uses Tailwind? → Install tailwindcss@^4.0.0
│  └─ ALWAYS use Tailwind v4 (CSS-first config, no tailwind.config.js)
│
├─ Need design tokens / theming? → CSS Modules + CSS custom properties in tokens.css
│
└─ Building a component library? → CSS Modules (maximum portability)
```

---

## Scaffolding Checklist

When generating a React project, ALWAYS complete every item:

1. [ ] Create `index.html` with `<div id="root"></div>` and `<script type="module" src="/src/main.tsx"></script>`
2. [ ] Create `vite.config.ts` with path aliases and `@vitejs/plugin-react`
3. [ ] Create `tsconfig.json` with strict mode and path aliases
4. [ ] Create `tsconfig.node.json` for build tool files
5. [ ] Create `src/main.tsx` with `createRoot` (React 18) or `createRoot` (React 19)
6. [ ] Create `src/app/App.tsx` (medium/large) or `src/components/App.tsx` (small)
7. [ ] Create routing setup if medium/large (`src/app/router.tsx`)
8. [ ] Create providers wrapper if medium/large (`src/app/providers.tsx`)
9. [ ] Create `test/setup.ts` with testing-library matchers
10. [ ] Create `eslint.config.js` with flat config format
11. [ ] Create `.prettierrc` with consistent formatting rules
12. [ ] Create `.gitignore` with node_modules, dist, .env, coverage
13. [ ] Create `.env.example` with `VITE_` prefixed variables
14. [ ] Create `package.json` with all dependencies and scripts
15. [ ] Verify no `@types/react` if React 19

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete scaffold output for small, medium, and large projects
- [references/patterns.md](references/patterns.md) -- Project structure patterns with rationale

### Official Sources

- https://vite.dev/guide/
- https://react.dev/learn/start-a-new-react-project
- https://react.dev/learn/typescript
- https://reactrouter.com/home
- https://tanstack.com/query/latest/docs/framework/react/overview
- https://zustand.docs.pmnd.rs/getting-started/introduction
- https://vitest.dev/guide/
- https://testing-library.com/docs/react-testing-library/intro
- https://eslint.org/docs/latest/use/configure/configuration-files
