# SolidJS Claude Skill Package — Way of Work

> 7-phase methodology, skill structure, content standards, and naming conventions.

## 7-Phase Methodology

### Phase 1: Infrastructure & Research Setup
- Create repository structure and core files
- Initialize all documentation (CLAUDE.md, ROADMAP.md, etc.)
- Set up skill category directories
- Define sources and requirements

### Phase 2: Vooronderzoek (Preliminary Research)
- Broad research across all SolidJS topics
- Identify React-to-SolidJS anti-pattern catalog
- Map SolidJS ecosystem (core, SolidStart, router, primitives)
- Document findings in `docs/research/vooronderzoek-solidjs.md`
- Output: high-level understanding, skill candidate list

### Phase 3: Deep Topic Research
- Per-skill deep-dive research
- Code example verification against official docs
- Anti-pattern documentation with corrections
- Output: `docs/research/topic-research/{skill-name}-research.md`

### Phase 4: Masterplan Creation
- Define complete skill list with priorities
- Map dependencies between skills
- Create batch execution plan (3 skills per batch)
- Output: `docs/masterplan/solidjs-masterplan.md`

### Phase 5: Skill Writing
- Write skills in batches of 3
- Each batch follows: write → validate → fix → accept
- Use meta-orchestrator pattern (P-002) for delegation
- Quality gate after every batch

### Phase 6: Validation & Cross-Review
- All skills pass validator-before-apply checklist (P-003)
- Cross-reference check: no contradictions between skills
- React contamination audit: ensure no React patterns leaked into examples
- Line count compliance check

### Phase 7: Publication & Release
- README.md finalized as landing page
- GitHub repository published
- Release tags created
- Agent Skills registry submission

---

## Skill Structure

Every skill lives in its own directory under `skills/source/`:

```
skills/source/solid-{category}-{topic}/
├── SKILL.md           # Main skill file (< 500 lines)
└── references/        # Supporting files (optional)
    ├── examples.md    # Extended code examples
    ├── api-table.md   # API reference tables
    └── patterns.md    # Pattern catalog
```

### SKILL.md Structure

```markdown
---
name: solid-{category}-{topic}
description: |
  Trigger words that activate this skill. Include technology name,
  action verbs, and specific API names.
---

# Skill Title

## Quick Reference
Brief, scannable summary of the most important patterns.

## Decision Tree
When to use what — flowchart-style guidance.

## Patterns

### Pattern Name
- When: condition
- Do: action
- Why: rationale
- Example: code

## Anti-Patterns
What NOT to do and why.

## Version Notes
SolidJS 1.x vs 2.x differences.

## Reference Links
Links to references/ files and official docs.
```

### YAML Frontmatter Rules
- `name`: MUST match directory name exactly
- `description`: MUST contain trigger words that Claude uses to match the skill
  - Include the technology name ("SolidJS")
  - Include action verbs ("create", "build", "fix", "debug")
  - Include specific API names ("createSignal", "createStore", "Show", "For")
  - Include problem descriptions ("reactivity not working", "props undefined")

---

## Content Standards

### Language
- English only (D-003)
- Deterministic: "ALWAYS", "NEVER", "MUST", "DO NOT"
- No hedging: never "you might want to", "it's a good practice to"
- Direct: "Use X" not "You should consider using X"

### Code Examples
- TypeScript + TSX only (D-006)
- ALWAYS show imports
- ALWAYS show the React anti-pattern AND the SolidJS correction
- Format:
  ```tsx
  // WRONG - React pattern, breaks SolidJS reactivity
  const { count } = props; // destructuring kills tracking

  // CORRECT - SolidJS pattern, preserves reactivity
  <div>{props.count}</div> // direct access maintains tracking
  ```

### Anti-Pattern Documentation Format
```markdown
### Anti-Pattern: [Name]
- **React pattern**: what developers coming from React will try
- **Why it fails**: technical explanation of the reactivity break
- **SolidJS fix**: the correct approach
- **Detection**: how to spot this in code review
```

---

## Naming Conventions

### Skill Names
- Format: `solid-{category}-{topic}`
- Category: `syntax`, `impl`, `errors`, `core`, or agent name
- Topic: descriptive, lowercase, hyphens for multi-word
- Examples:
  - `solid-syntax-signals` — createSignal, createMemo, createEffect
  - `solid-syntax-stores` — createStore, produce, reconcile
  - `solid-syntax-jsx` — JSX compilation, control flow, event handling
  - `solid-impl-components` — component patterns, props, children
  - `solid-impl-routing` — solid-router, file-based routing
  - `solid-impl-solidstart` — SolidStart meta-framework
  - `solid-errors-react-contamination` — React anti-pattern catalog
  - `solid-errors-reactivity` — debugging reactivity breaks
  - `solid-core-overview` — API surface, version matrix
  - `solid-core-reactivity-model` — fine-grained reactivity internals

### File Names
- SKILL.md: always uppercase
- references/: always lowercase with hyphens
- Research docs: `{skill-name}-research.md`

---

## Batch Execution Strategy

### Batch Size
- 3 skills per batch (optimal for Claude Code Agent tool)
- NEVER two agents working on the same file

### Batch Order (Recommended)
1. **Core first**: solid-core-* (foundational knowledge)
2. **Syntax second**: solid-syntax-* (API patterns)
3. **Implementation third**: solid-impl-* (workflows)
4. **Errors fourth**: solid-errors-* (builds on all above)
5. **Agents last**: solid-agents-* (needs all skills as reference)

### Quality Gate
After every batch:
1. Run validator-before-apply checklist
2. Check for React pattern contamination
3. Verify cross-references between skills
4. Update ROADMAP.md progress
