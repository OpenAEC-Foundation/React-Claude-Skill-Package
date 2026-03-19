# Way of Work — React Claude Skill Package

## 7-Phase Methodology

### Phase 1: Infrastructure Setup
- Initialize repository with core files
- Define directory structure, protocols, and conventions
- Establish quality criteria and source verification rules

### Phase 2: Vooronderzoek (Broad Research)
- Survey the React 18/19 landscape across all skill categories
- Identify all skill topics with priorities
- Map React API surface area (hooks, components, DOM, server)
- Output: `docs/research/vooronderzoek-react.md`

### Phase 3: Deep Dives (Targeted Research)
- Per-topic research producing detailed analysis
- Verify code examples against official documentation via WebFetch
- Identify anti-patterns from GitHub issues
- Output: `docs/research/topic-research/{skill-name}-research.md`

### Phase 4: Skill Architecture
- Define skill boundaries, dependencies, cross-references
- Create skill dependency graph
- Finalize skill templates and agent prompts
- Ensure no overlapping content between skills

### Phase 5: Skill Creation
- Agent-based batch workflow (3 agents per batch)
- Each agent writes one skill with complete content
- Cross-validation between agents
- Quality gate after every batch

### Phase 6: Validation
- Automated format checks (line count, frontmatter, language)
- Content accuracy validation against official docs
- Cross-reference link verification
- Technical accuracy review (Rules of Hooks, TypeScript types)

### Phase 7: Publication
- README.md finalization
- CHANGELOG.md completion
- GitHub repository publication
- Social preview and branding

---

## Skill Structure

### Directory Layout
```
skills/source/react-{category}/react-{category}-{topic}/
├── SKILL.md              # Entry point (< 500 lines)
└── references/           # Extended content (optional)
    ├── examples.md       # Detailed code examples
    ├── api-table.md      # API reference tables
    ├── migration.md      # Version migration notes
    └── patterns.md       # Extended pattern catalog
```

### SKILL.md Template
```markdown
---
name: react-{category}-{topic}
description: |
  Trigger words and description for Claude to match this skill.
  Include: what it covers, when to use it, key concepts.
---

# React {Topic Title}

## Quick Reference
<!-- Most common patterns, copy-paste ready -->
<!-- 3-5 essential code snippets -->

## Decision Tree
<!-- When X, use Y -->
<!-- Branching logic for common decisions -->

## Patterns

### Pattern Name
<!-- Context: when to use -->
<!-- Implementation: code example -->
<!-- Notes: version differences, gotchas -->

## Anti-Patterns
<!-- What NOT to do and WHY -->

## Version Notes
<!-- React 18 vs 19 differences -->
<!-- Migration guidance -->

## References
- [Extended examples](references/examples.md)
- [Official docs](https://react.dev/reference/react/...)
```

### YAML Frontmatter Requirements
```yaml
---
name: react-{category}-{topic}    # Must match directory name
description: |
  Trigger words that help Claude match user queries to this skill.
  MUST include: technology (React), topic area, key concepts.
  Example: "React hooks useState useEffect state management
  component lifecycle side effects cleanup dependencies"
---
```

---

## Naming Conventions

### Skill Naming
- Format: `react-{category}-{topic}`
- Category: syntax, impl, errors, core, agents
- Topic: kebab-case, descriptive, concise
- Examples:
  - `react-syntax-hooks` — Hooks API patterns
  - `react-syntax-jsx` — JSX/TSX patterns and expressions
  - `react-impl-testing` — Testing with React Testing Library
  - `react-errors-boundaries` — Error boundary patterns
  - `react-core-rendering` — Rendering model and reconciliation

### File Naming
- SKILL.md — Always uppercase
- references/ — Always lowercase directory
- Reference files — lowercase kebab-case: `examples.md`, `api-table.md`

---

## Content Standards

### Code Examples
- ALWAYS TypeScript/TSX
- ALWAYS include type annotations
- ALWAYS show imports
- ALWAYS show the component/hook signature with types
- Keep examples focused — one concept per example
- Use realistic names (not `foo`, `bar`, `MyComponent`)

### Language
- English only in all skill content
- Deterministic: "ALWAYS", "NEVER", "MUST", "MUST NOT"
- No hedging: avoid "you might", "consider", "perhaps"
- Action-oriented: tell Claude what to DO

### Version Handling
```markdown
<!-- For React 19-only features -->
> **React 19+**: This feature requires React 19 or later.

<!-- For version differences -->
### React 18
\`\`\`tsx
// React 18 approach
\`\`\`

### React 19
\`\`\`tsx
// React 19 approach — preferred when available
\`\`\`
```

---

## Agent Workflow

### Batch Composition
- 3 agents per batch
- Each agent works on a SEPARATE skill (never same file)
- Agent prompt includes:
  - Skill name and category
  - Relevant REQUIREMENTS.md sections
  - Approved SOURCES.md URLs
  - Constraints from DECISIONS.md
  - Skill template from this file

### Agent Prompt Template
```
You are creating a React skill for the Claude Skill Package.

**Skill**: react-{category}-{topic}
**Category**: {category}
**Purpose**: {what this skill teaches Claude}

**Constraints**:
- English only (D-001)
- SKILL.md < 500 lines (D-005)
- TypeScript/TSX code examples (D-006)
- React 18 + 19 coverage (D-004)
- Deterministic language (REQUIREMENTS.md)

**Sources** (use ONLY these):
- {relevant URLs from SOURCES.md}

**Structure** (follow exactly):
1. YAML frontmatter with name + description
2. Quick Reference
3. Decision Tree
4. Patterns
5. Anti-Patterns
6. Version Notes
7. References

**Output**: Complete SKILL.md file content
```

### Quality Gate
After each batch:
1. Validate all 3 skills against REQUIREMENTS.md checklist
2. Check line counts (< 500)
3. Verify English-only
4. Confirm TypeScript types in all examples
5. Check version coverage notes
6. Accept or respawn with corrections
