# Anti-Patterns (React Project Setup)

## 1. Using Create React App

```bash
# WRONG: CRA is deprecated and unmaintained
npx create-react-app my-app --template typescript

# CORRECT: Use Vite with react-ts template
npm create vite@latest my-app -- --template react-ts
```

**WHY**: Create React App (CRA) was officially deprecated by the React team. It uses Webpack with slow build times, has outdated dependencies with security vulnerabilities, and receives no updates. Vite provides faster dev startup, instant HMR, and active maintenance.

---

## 2. Disabling TypeScript Strict Mode

```jsonc
// WRONG: Disabling strict mode to "fix" type errors
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": false
  }
}

// CORRECT: Enable strict mode and fix the actual type issues
{
  "compilerOptions": {
    "strict": true
  }
}
```

**WHY**: Disabling strict mode hides real bugs. `noImplicitAny: false` allows untyped values to slip through as `any`, defeating the purpose of TypeScript. Fix the type errors instead of suppressing them.

---

## 3. Forgetting isolatedModules

```jsonc
// WRONG: Missing isolatedModules breaks Vite transpilation
{
  "compilerOptions": {
    "isolatedModules": false
  }
}

// CORRECT: Required for Vite
{
  "compilerOptions": {
    "isolatedModules": true
  }
}
```

**WHY**: Vite uses esbuild for transpilation, which processes each file independently. Without `isolatedModules: true`, TypeScript allows patterns (like `const enum` across files) that break when files are transpiled in isolation.

---

## 4. Path Aliases in Only One Config

```typescript
// WRONG: Alias only in vite.config.ts -- TypeScript cannot resolve imports
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
// tsconfig.app.json -- missing paths! TypeScript shows red squiggles.
```

```jsonc
// CORRECT: Configure in BOTH files
// tsconfig.app.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

```typescript
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
```

**WHY**: TypeScript and Vite have separate module resolution. TypeScript uses `paths` for type checking and editor IntelliSense. Vite uses `resolve.alias` for bundling. Both must be configured for the full toolchain to work.

---

## 5. Exposing Secrets in Environment Variables

```bash
# WRONG: Sensitive key with VITE_ prefix -- bundled into client code!
VITE_DATABASE_URL=postgres://user:password@host/db
VITE_SECRET_KEY=sk-live-abc123

# CORRECT: Only expose public values with VITE_ prefix
VITE_API_URL=https://api.example.com
DATABASE_URL=postgres://user:password@host/db  # NOT exposed to client
```

**WHY**: Any variable prefixed with `VITE_` is statically replaced in the client bundle and visible to anyone who inspects the JavaScript. API keys, database URLs, and secrets MUST use unprefixed names so they remain server-side only.

---

## 6. Missing Type Declarations for Env Vars

```typescript
// WRONG: Using import.meta.env without type declarations
const url = import.meta.env.VITE_API_URL; // type: any

// CORRECT: Declare types in env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}

const url = import.meta.env.VITE_API_URL; // type: string
```

**WHY**: Without type declarations, all `import.meta.env` values are typed as `any`. You lose autocompletion, typo detection, and type safety. A misspelled variable name silently returns `undefined` at runtime.

---

## 7. Committing .env.local Files

```gitignore
# WRONG: .gitignore missing local env files
node_modules/
dist/

# CORRECT: Always ignore local env overrides
node_modules/
dist/
.env.local
.env.*.local
```

**WHY**: `.env.local` and `.env.*.local` files contain developer-specific overrides and may include secrets. Committing them exposes credentials and causes conflicts between developers.

---

## 8. Using .eslintrc (Legacy Format)

```javascript
// WRONG: Legacy .eslintrc.js format (deprecated in ESLint v9)
module.exports = {
  extends: ['react-app'],
  rules: { /* ... */ },
};

// CORRECT: Flat config format (eslint.config.js)
import js from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['dist'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
  },
);
```

**WHY**: The `.eslintrc.*` configuration format is deprecated. ESLint 9+ uses the flat config format (`eslint.config.js`). The legacy format will be removed in a future major version.

---

## 9. Skipping type-check in CI/Build

```jsonc
// WRONG: Build without type checking
{
  "scripts": {
    "build": "vite build"
  }
}

// CORRECT: Type-check before building
{
  "scripts": {
    "build": "tsc -b && vite build",
    "type-check": "tsc -b --noEmit"
  }
}
```

**WHY**: Vite uses esbuild for transpilation, which strips types but does NOT check them. Without an explicit `tsc` step, type errors slip through to production. ALWAYS run `tsc -b` before `vite build`.

---

## 10. Deep Relative Import Paths

```typescript
// WRONG: Fragile deep relative imports
import { Button } from '../../../components/ui/Button';
import { useAuth } from '../../../../hooks/useAuth';
import { formatDate } from '../../../utils/formatDate';

// CORRECT: Path aliases for clean, refactor-safe imports
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
import { formatDate } from '@/utils/formatDate';
```

**WHY**: Deep relative paths break when files move, are hard to read, and make it unclear where in the project tree a file lives. Path aliases (`@/`) provide stable, readable imports that survive refactoring.

---

## 11. Storing Tests in a Separate Directory Tree

```
# WRONG: Tests separated from source
src/
├── components/
│   └── Button.tsx
tests/
├── components/
│   └── Button.test.tsx

# CORRECT: Tests co-located with source
src/
├── components/
│   ├── Button.tsx
│   └── Button.test.tsx
```

**WHY**: Separate test directories create a parallel file tree that must be manually kept in sync. When a component moves or is renamed, its test is easily forgotten. Co-location ensures tests move with their source and makes test coverage gaps immediately visible.

---

## 12. Using jsx: "react" Instead of "react-jsx"

```jsonc
// WRONG: Legacy JSX transform (requires "import React" in every file)
{
  "compilerOptions": {
    "jsx": "react"
  }
}

// CORRECT: Automatic JSX transform (React 17+)
{
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```

**WHY**: The `"react"` JSX mode uses the classic `React.createElement` transform, requiring `import React from 'react'` in every file that uses JSX. The `"react-jsx"` mode uses the automatic transform introduced in React 17, which inserts the import automatically. This reduces boilerplate and slightly improves bundle size.
