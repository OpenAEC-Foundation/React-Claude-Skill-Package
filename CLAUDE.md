# React Claude Skill Package

## Standing Orders — READ THIS FIRST

**Mission**: Build a complete, production-ready skill package for React and publish it under the OpenAEC Foundation on GitHub. This is your standing order for every session in this workspace.

**How**: Follow the 7-phase research-first methodology. Delegate ALL execution to agents. You are the ARCHITECT — you think, plan, validate, and delegate. Agents do the actual work.

**What you do on session start**:
1. Read ROADMAP.md → determine current phase and next steps
2. Read all core files (LESSONS.md, DECISIONS.md, REQUIREMENTS.md, SOURCES.md)
3. Continue where the previous session left off
4. If Phase 1 is incomplete → create the raw masterplan first
5. If Phase 2+ → follow the methodology, delegating in batches of 3 agents

**Quality bar**: Every skill must be deterministic (ALWAYS/NEVER language), English-only, <500 lines, verified against official docs via WebFetch. No hallucinated APIs. No vague language.

**End state**: A published GitHub repo at `https://github.com/OpenAEC-Foundation/React-Claude-Skill-Package` with:
- All skills created, validated, and organized
- INDEX.md with complete skill catalog
- README.md with installation instructions and skill table
- Social preview banner (1280x640px) with OpenAEC branding
- Release tag (v1.0.0) and GitHub release
- Repository topics set (claude, skills, react, frontend, ui, ai, deterministic, openaec)

**Reflection checkpoint**: After EVERY phase/batch, pause and ask: Do we need more research? Should we revise the plan? Are we meeting quality standards? Update core files before proceeding.

**Consolidate lessons**: Any workflow-level insight (not tech-specific) should also be noted for consolidation back to the Workflow Template repo (`C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template`).

**Masterplan template**: When creating your masterplan in Phase 3, follow the EXACT structure from:
- Template: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\templates\masterplan.md.template`
- Proven example: `C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\docs\masterplan\tauri-masterplan.md` (27 skills, 10 batches, executed in one session)

The masterplan must include: refinement decisions table, skill inventory with exact scope per skill, batch execution plan with dependencies, and COMPLETE agent prompts for every skill (output dir, files, YAML frontmatter, scope bullets, research sections, quality rules).

**Reference projects** (study these for methodology, not content):
- ERPNext (28 skills): https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
- Blender-Bonsai (73 skills): https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
- Tauri 2 (27 skills): https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package

---

## Project Identity
- React skill package for Claude — single technology, frontend UI library
- Technology: React 18.x and React 19.x (UI component library)
- Focus: Declarative components, hooks, Server Components, concurrent features, React Compiler
- Languages: TypeScript, JSX/TSX
- Methodology: 7-phase research-first development (proven in ERPNext, Blender, and Tauri packages)
- Reference projects:
  - ERPNext: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
  - Tauri: https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package
  - Blender-Bonsai: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package

## Core Files Map
| File | Domain | Role |
|------|--------|------|
| ROADMAP.md | Status | Single source of truth for project status, progress, next steps |
| LESSONS.md | Knowledge | Numbered lessons (L-XXX) discovered during development |
| DECISIONS.md | Architecture | Numbered decisions (D-XXX) with rationale, immutable once recorded |
| REQUIREMENTS.md | Scope | What skills must achieve, quality guarantees |
| SOURCES.md | References | Official documentation URLs, verification rules, last-verified dates |
| WAY_OF_WORK.md | Methodology | 7-phase process, skill structure, content standards |
| CHANGELOG.md | History | Version history in Keep a Changelog format |
| docs/masterplan/react-masterplan.md | Planning | Execution plan with phases, prompts, dependencies |
| README.md | Public | GitHub landing page |

## Technology Scope
| Tech | Prefix | Versions |
|------|--------|----------|
| React | react- | v18.x, v19.x |

## Skill Categories
| Category | Purpose | Naming |
|----------|---------|--------|
| syntax/ | Hooks API, component patterns, JSX, event handling | react-syntax-{topic} |
| impl/ | Development workflows, testing, performance | react-impl-{topic} |
| errors/ | Error boundaries, debugging, anti-patterns | react-errors-{topic} |
| core/ | Architecture, state management, rendering model | react-core-{topic} |
| agents/ | Intelligent orchestration, code generation | react-{agent-name} |

## Repository Structure
```
project-root/
├── CLAUDE.md                    # THIS FILE - protocols and instructions
├── ROADMAP.md                   # Status (single source of truth)
├── REQUIREMENTS.md              # Quality guarantees
├── DECISIONS.md                 # Architectural decisions
├── SOURCES.md                   # Official reference URLs
├── WAY_OF_WORK.md               # 7-phase methodology
├── LESSONS.md                   # Lessons learned
├── CHANGELOG.md                 # Version history
├── README.md                    # GitHub landing page
├── docs/
│   ├── masterplan/              # react-masterplan.md
│   └── research/                # vooronderzoek-react.md, topic-research/, fragments/
└── skills/
    └── source/
        ├── react-syntax/        # Hooks, JSX, component API, event handling
        ├── react-impl/          # Testing, performance, SSR, build workflows
        ├── react-errors/        # Error boundaries, debugging, anti-patterns
        ├── react-core/          # Architecture, rendering model, state management
        └── react-agents/        # Validation, code generation agents
```
Single technology package — all skills share the `react-` prefix.

---

## Session Start Protocol (P-001)
EVERY session begins with this sequence:

1. **Read ROADMAP.md** → Determine current phase, progress percentage, and "Next Steps" section
2. **Read LESSONS.md** → Check recent lessons that may affect your work
3. **Read DECISIONS.md** → Know all architectural decisions (D-001+) and their constraints
4. **Read REQUIREMENTS.md** → Understand quality guarantees and per-area requirements
5. **Read docs/masterplan/react-masterplan.md** → Know the execution plan and current phase details
6. **If researching**: Read **SOURCES.md** → Know approved sources, verification rules
7. **If creating skills**: Read **WAY_OF_WORK.md** → Know skill structure, content standards, naming
8. Identify next action from ROADMAP.md "Next Steps"
9. Confirm with user before proceeding

---

## Meta-Orchestrator Protocol (P-002)

### Identity
This Claude Code session + the human user together ARE the **meta-orchestrator**.
We are NOT a relay/passthrough. We are the strategic brain.

**What we do HERE (the brain):**
- THINK: Analyze problems, design solutions, make architectural decisions
- STRATEGIZE: Plan agent batches, define task decomposition, choose approaches
- DECIDE: Accept/reject agent output, resolve conflicts, set direction
- COMPOSE: Craft precise agent prompts with full context from core files

**What agents do THERE (the hands):**
- EXECUTE: Research, write, code, validate — the actual work
- CROSS-VALIDATE: Agents check each other's output before it comes back to us
- REPORT: Deliver refined, verified output to the meta-orchestrator

### Rules:
- Delegate EXECUTION via Claude Code Agent tool — thinking stays here
- Validate before accepting (validator-before-apply)
- Strategic reasoning, planning, and decision-making happen in THIS session
- Agents receive complete context (core file references) so they can work autonomously

### What to include in EVERY agent prompt:
- Quality criteria from **REQUIREMENTS.md** (relevant to their task)
- Approved source URLs from **SOURCES.md** (what docs to consult)
- Current status from **ROADMAP.md** (what's done, what's needed)
- Relevant constraints from **DECISIONS.md** (D-003: English-only, etc.)
- Skill structure from **WAY_OF_WORK.md** (if writing skills)

### Delegation Flow:
1. **Think** — Define task scope, expected output, success criteria
2. **Compose** — Write task prompt with core file references (see above)
3. **Spawn Agent** — Use Claude Code Agent tool with complete prompt
4. **Collect** — Receive agent output automatically
5. **Judge** — VALIDATE output against **REQUIREMENTS.md** quality criteria
6. **Iterate** — Accept, or respawn with corrections

### Batch Strategy:
- 3 agents per batch (optimal for Claude Code Agent tool)
- Separated file scopes (NEVER two agents on same file)
- Quality gate after every batch
- Cross-validation: review agent output before final acceptance

---

## Quality Control Protocol (P-003)
### Validation criteria sourced from core files:

**From REQUIREMENTS.md:**
- Skill format requirements (YAML frontmatter, structure)
- React 18/19 version coverage
- TypeScript/TSX code examples required
- Hooks patterns must follow Rules of Hooks

**From DECISIONS.md:**
- D-001: English-only content
- D-002: MIT License
- D-004: React 18 + 19 dual version coverage
- D-005: SKILL.md < 500 lines

**From SOURCES.md:**
- All code verified against listed official sources only
- No unverified blog posts or outdated content

### Validator-Before-Apply checklist:
1. File exists and is complete
2. YAML frontmatter valid (name, description with trigger words)
3. Line count < 500 (SKILL.md)
4. English-only (no Dutch or other languages)
5. Deterministic language (ALWAYS/NEVER, not "you might consider")
6. React 18 + React 19 version differences noted where applicable
7. All references/ files exist and are linked from SKILL.md
8. Sources traceable to **SOURCES.md** approved URLs
9. Hooks follow Rules of Hooks (top-level only, React functions only)
10. TypeScript types included in all code examples

### Correction Flow:
If validation fails:
1. Document what failed in agent feedback
2. Spawn fix-agent with specific correction instructions
3. Re-validate after fix
4. NEVER accept below quality bar defined in **REQUIREMENTS.md**

---

## Research Protocol (P-004)
### Before ANY research:
1. Read **SOURCES.md** → Know approved sources for React
2. Read **REQUIREMENTS.md** → Know what the research must cover
3. Read **DECISIONS.md** → Know constraints (D-001 English-only, D-004 React 18+19, D-007 WebFetch, etc.)

### During research:
- Use ONLY sources listed in **SOURCES.md** (or add new ones there)
- Verify code examples against official documentation
- Identify anti-patterns from real GitHub issues
- Use WebFetch to ensure latest documentation is consulted (D-007)
- Pay special attention to React 18 → 19 migration differences

### After research:
1. Update **SOURCES.md** "Last Verified" table with verification date
2. Log new discoveries in **LESSONS.md** (numbered L-XXX)
3. If new architectural decisions emerge, record in **DECISIONS.md** (numbered D-XXX)

### Research output location:
- Vooronderzoek: `docs/research/vooronderzoek-react.md`
- Topic research: `docs/research/topic-research/{skill-name}-research.md`
- Research fragments: `docs/research/fragments/`

---

## Skill Standards (P-005)
Defined in detail in **WAY_OF_WORK.md** and **REQUIREMENTS.md**. Quick reference:

- English-only (per **DECISIONS.md** D-001)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- SKILL.md < 500 lines (per **DECISIONS.md** D-005), heavy content in references/
- YAML frontmatter: name + description with trigger words
- Structure: Quick Reference > Decision Trees > Patterns > Reference Links
- Naming: `react-{category}-{topic}`
- Version coverage: Note React 18/19 differences where applicable (per D-004)
- TypeScript types in all code examples
- Verify against **SOURCES.md** approved URLs only

---

## Document Sync Protocol (P-006)
After EVERY completed phase/batch, update these files:

1. **ROADMAP.md** → Status, percentage, changelog entry, next steps (MANDATORY)
2. **LESSONS.md** → New patterns or discoveries (if any)
3. **DECISIONS.md** → New architectural decisions (if any)
4. **SOURCES.md** → New sources verified or dates updated (if researching)
5. **CHANGELOG.md** → Milestone entries (for significant completions)
6. Commit with message: `Phase X.Y: [action] [subject]`
7. **README.md** → Check if landing page needs updating:
   - Skill count changed? Update package table
   - Phase milestone reached? Update "Current Progress" section
   - New documentation added? Update docs table

Timing: IMMEDIATE after completion, not deferred.

---

## Session End Protocol (P-007)
Before ending ANY session:

1. **ROADMAP.md** → Update current phase status + "Next Steps" section (CRITICAL - this is how the next session knows where to continue)
2. **LESSONS.md** → Log anything learned during this session
3. **DECISIONS.md** → Record any decisions made
4. **CHANGELOG.md** → Add entry if milestone reached
5. Commit all changes with descriptive message
6. Verify **README.md** reflects current project state

---

## Inter-Agent Protocol (P-008)
Not applicable in the traditional sense — this project uses Claude Code Agent tool, not oa-cli.

Agents are spawned via the Agent tool within Claude Code. Results are collected automatically when the agent completes. No explicit messaging protocol needed.

### How it works:
- Meta-orchestrator composes a prompt with full context
- Agent tool spawns a subagent with that prompt
- Subagent executes and returns results
- Meta-orchestrator validates output against REQUIREMENTS.md
- Accept or respawn with corrections

---

## GitHub Publication Protocol (P-009)
### Repository: OpenAEC-Foundation/React-Claude-Skill-Package

### Publication checklist:
1. All core files present and up-to-date
2. README.md serves as effective landing page
3. LICENSE file present (MIT)
4. .gitignore configured
5. No sensitive data in repository
6. Skills validated against REQUIREMENTS.md
7. CHANGELOG.md reflects current state

### Branch strategy:
- `main` — stable, validated content only
- Feature branches for development work
- PR-based workflow for significant changes
