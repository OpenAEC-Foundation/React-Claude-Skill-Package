# Sources — React Claude Skill Package

## Verification Rules
1. ONLY use sources listed in this file for skill content
2. When adding a new source, add it here FIRST with category and URL
3. Update "Last Verified" date when you verify content against a source
4. Blog posts and unofficial tutorials are NOT acceptable as primary sources
5. GitHub issues/discussions may be used for anti-pattern identification only

---

## Primary Sources

### Official Documentation
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| React Documentation | https://react.dev | Primary reference | 2026-03-19 |
| React API Reference | https://react.dev/reference/react | API signatures | 2026-03-19 |
| React DOM API Reference | https://react.dev/reference/react-dom | DOM API signatures | 2026-03-19 |
| React Learn | https://react.dev/learn | Official tutorials | 2026-03-19 |
| React Blog | https://react.dev/blog | Release notes, RFCs | 2026-03-19 |

### Source Code & Repository
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| React GitHub Repository | https://github.com/facebook/react | Source code | 2026-03-19 |
| React RFC Repository | https://github.com/reactjs/rfcs | Design decisions | 2026-03-19 |
| React Types (@types/react) | https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/react | TypeScript types | 2026-03-19 |

### React 19 Specific
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| React 19 Release Notes | https://react.dev/blog/2024/12/05/react-19 | Release announcement | 2026-03-19 |
| React Compiler | https://react.dev/learn/react-compiler | Compiler docs | 2026-03-19 |
| Server Components | https://react.dev/reference/rsc/server-components | RSC reference | 2026-03-19 |

### Ecosystem — Testing
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| React Testing Library | https://testing-library.com/docs/react-testing-library/intro | Testing patterns | 2026-03-19 |
| Vitest | https://vitest.dev | Test runner | 2026-03-19 |

### Ecosystem — Build & Tooling
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| Vite | https://vite.dev | Build tool | 2026-03-19 |
| Next.js Documentation | https://nextjs.org/docs | React framework | 2026-03-19 |
| React Router | https://reactrouter.com | Routing | 2026-03-19 |

### Ecosystem — State Management
| Source | URL | Category | Last Verified |
|--------|-----|----------|---------------|
| Zustand | https://zustand-demo.pmnd.rs | State management | 2026-03-19 |
| TanStack Query | https://tanstack.com/query | Server state | 2026-03-19 |

---

## Source Usage Guidelines

### For Hooks & Component API
- Primary: react.dev/reference/react
- Verify signatures and behavior against official API reference
- Check @types/react for TypeScript type definitions

### For Patterns & Best Practices
- Primary: react.dev/learn
- Cross-reference with react.dev/reference for accuracy
- Use GitHub issues for anti-pattern identification

### For React 19 Features
- Primary: React 19 release notes + react.dev/reference
- Compiler: react.dev/learn/react-compiler
- Server Components: react.dev/reference/rsc/server-components
- ALWAYS mark React 19-only features with version badges

### For Testing
- Primary: React Testing Library documentation
- NEVER reference Enzyme (deprecated, incompatible with React 18+)
- Use Vitest as preferred test runner (not Jest, unless specifically needed)

### For SSR/Frameworks
- Reference Next.js docs for Server Component patterns
- Distinguish between React core features and framework-specific features
