# SolidJS Claude Skill Package

## Project Identity
- SolidJS skill package for Claude — reactive UI framework with fine-grained reactivity
- Technology: SolidJS 1.x / 2.x (reactive UI framework, no Virtual DOM)
- Core problem: React anti-pattern contamination — Claude systematically generates React patterns that silently break SolidJS reactivity
- Languages: TypeScript, JSX/TSX
- Methodology: 7-phase research-first development (proven in ERPNext, Blender, and Tauri packages)
- Reference projects:
  - ERPNext: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
  - Blender-Bonsai: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
  - Tauri 2: https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package

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
| docs/masterplan/solidjs-masterplan.md | Planning | Execution plan with phases, prompts, dependencies |
| README.md | Public | GitHub landing page |

## Technology Scope
| Tech | Prefix | Versions |
|------|--------|----------|
| SolidJS | solid- | 1.x, 2.x |

## Skill Categories
| Category | Purpose | Naming |
|----------|---------|--------|
| syntax/ | Signals, stores, reactivity primitives, JSX compilation | solid-syntax-{topic} |
| impl/ | Component patterns, routing, SolidStart, state management | solid-impl-{topic} |
| errors/ | React contamination detection, reactivity debugging | solid-errors-{topic} |
| core/ | Architecture, fine-grained reactivity model, version matrix | solid-core-{topic} |
| agents/ | Validation, React-pattern detection, code generation | solid-{agent-name} |

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
│   ├── masterplan/              # solidjs-masterplan.md
│   └── research/                # vooronderzoek-solidjs.md, topic-research/, fragments/
└── skills/
    └── source/
        ├── solid-syntax/        # Signals, stores, effects, JSX, reactivity primitives
        ├── solid-impl/          # Component patterns, routing, SolidStart workflows
        ├── solid-errors/        # React anti-patterns, reactivity breaks, debugging
        ├── solid-core/          # API overview, reactivity model, version matrix
        └── solid-agents/        # Validation, React-pattern detection agents
```
Single technology package — all skills share the `solid-` prefix.

---

## Session Start Protocol (P-001)
EVERY session begins with this sequence:

1. **Read ROADMAP.md** → Determine current phase, progress percentage, and "Next Steps" section
2. **Read LESSONS.md** → Check recent lessons that may affect your work
3. **Read DECISIONS.md** → Know all architectural decisions (D-001+) and their constraints
4. **Read REQUIREMENTS.md** → Understand quality guarantees and per-area requirements
5. **Read docs/masterplan/solidjs-masterplan.md** → Know the execution plan and current phase details
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
- Relevant constraints from **DECISIONS.md** (D-003: English-only, D-006: TypeScript+TSX coverage, etc.)
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
- SolidJS 1.x and 2.x version coverage
- TypeScript + TSX code examples mandatory

**From DECISIONS.md:**
- D-003: English-only content
- D-005: MIT License
- D-006: TypeScript + TSX coverage (no plain JS examples)
- D-007: SKILL.md < 500 lines

**From SOURCES.md:**
- All code verified against listed official sources only
- No unverified blog posts or outdated content

### Validator-Before-Apply checklist:
1. File exists and is complete
2. YAML frontmatter valid (name, description with trigger words)
3. Line count < 500 (SKILL.md)
4. English-only (no Dutch or other languages)
5. Deterministic language (ALWAYS/NEVER, not "you might consider")
6. TypeScript + TSX code examples (no plain JS)
7. NO React patterns — all examples must use SolidJS idioms
8. All references/ files exist and are linked from SKILL.md
9. Sources traceable to **SOURCES.md** approved URLs

### Correction Flow:
If validation fails:
1. Document what failed in agent feedback
2. Spawn fix-agent with specific correction instructions
3. Re-validate after fix
4. NEVER accept below quality bar defined in **REQUIREMENTS.md**

---

## Research Protocol (P-004)
### Before ANY research:
1. Read **SOURCES.md** → Know approved sources for SolidJS
2. Read **REQUIREMENTS.md** → Know what the research must cover
3. Read **DECISIONS.md** → Know constraints (D-003 English-only, D-004 SolidJS 1.x+2.x, etc.)

### During research:
- Use ONLY sources listed in **SOURCES.md** (or add new ones there)
- Verify code examples against official documentation
- Identify React anti-patterns and their SolidJS corrections
- Use WebFetch to ensure latest documentation is consulted
- Pay special attention to reactivity system differences from React

### After research:
1. Update **SOURCES.md** "Last Verified" table with verification date
2. Log new discoveries in **LESSONS.md** (numbered L-XXX)
3. If new architectural decisions emerge, record in **DECISIONS.md** (numbered D-XXX)

### Research output location:
- Vooronderzoek: `docs/research/vooronderzoek-solidjs.md`
- Topic research: `docs/research/topic-research/{skill-name}-research.md`
- Research fragments: `docs/research/fragments/`

---

## Skill Standards (P-005)
Defined in detail in **WAY_OF_WORK.md** and **REQUIREMENTS.md**. Quick reference:

- English-only (per **DECISIONS.md** D-003)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- SKILL.md < 500 lines (per **DECISIONS.md** D-007), heavy content in references/
- YAML frontmatter: name + description with trigger words
- Structure: Quick Reference > Decision Trees > Patterns > Reference Links
- Naming: `solid-{category}-{topic}`
- TypeScript + TSX only — no plain JavaScript examples (per D-006)
- EVERY code example must be valid SolidJS — no React patterns allowed
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

### Repository
- Organization: OpenAEC-Foundation
- Repository: SolidJS-Claude-Skill-Package
- URL: https://github.com/OpenAEC-Foundation/SolidJS-Claude-Skill-Package
- License: MIT (D-005)

### Publication checklist:
1. All core files present and up-to-date
2. README.md serves as effective landing page
3. LICENSE file present (MIT)
4. .gitignore configured
5. No secrets, credentials, or personal data
6. All skill YAML frontmatter valid
7. Social preview image set (if applicable)
