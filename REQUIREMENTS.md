# SolidJS Claude Skill Package — Requirements

> Quality guarantees and per-area requirements for all skills in this package.

## Global Requirements

### Format
- EVERY skill has a `SKILL.md` file with valid YAML frontmatter
- YAML frontmatter MUST include: `name`, `description` (with trigger words)
- SKILL.md MUST be < 500 lines (heavy content goes in `references/`)
- English-only content (no Dutch or other languages)
- Deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- No hedging: never "you might consider", "it's generally a good idea"

### Code Examples
- ALL code examples in TypeScript + TSX (no plain JavaScript)
- EVERY example MUST be valid SolidJS — no React patterns allowed
- ALWAYS show the wrong (React) pattern AND the correct (SolidJS) pattern side by side
- Include import statements in examples
- Show both the anti-pattern and the fix for error-related skills

### Version Coverage
- Skills MUST cover SolidJS 1.x (current stable)
- Skills MUST note SolidJS 2.x changes where applicable
- Version-specific behavior MUST be clearly marked with version badges
- SolidStart coverage where the topic involves meta-framework features

### Source Verification
- ALL code verified against sources listed in SOURCES.md
- No unverified blog posts, Stack Overflow answers, or outdated tutorials
- When in doubt, verify against the SolidJS source code on GitHub

---

## Per-Area Requirements

### Signals & Reactivity (solid-syntax-*)
- MUST explain fine-grained reactivity model (no VDOM diffing)
- MUST cover: createSignal, createMemo, createEffect, createResource
- MUST show signal access patterns (function call `count()` vs React's `count`)
- MUST warn about destructuring (kills reactivity tracking)
- MUST explain reactive scope and tracking context
- MUST cover batch updates and untrack()
- MUST differentiate from React's useState/useEffect mental model

### Stores & State (solid-syntax-*, solid-impl-*)
- MUST cover createStore and its proxy-based reactivity
- MUST explain nested reactivity (deep tracking by default)
- MUST show produce() for immutable-style updates
- MUST cover reconcile() for replacing store data
- MUST warn about direct mutation vs setter function patterns

### JSX & Components (solid-syntax-*, solid-impl-*)
- MUST explain SolidJS JSX compilation (components run once, not on every render)
- MUST cover control flow components: Show, For, Switch/Match, Index, Dynamic
- MUST warn about Array.map() — ALWAYS use `<For>` instead
- MUST explain props handling (no destructuring, use splitProps/mergeProps)
- MUST cover children() helper for working with children props
- MUST show event handling differences from React (onclick vs onClick)

### SolidStart (solid-impl-*)
- MUST cover file-based routing
- MUST explain server functions (server$)
- MUST cover data loading (createRouteData / useRouteData patterns)
- MUST address SSR and hydration
- MUST show API route patterns
- MUST note SolidStart version differences (0.x vs 1.x)

### Error Handling & Anti-Patterns (solid-errors-*)
- MUST catalog React-to-SolidJS pattern contamination
- MUST provide detection rules (what to look for in code)
- MUST show the fix for every anti-pattern
- MUST include ErrorBoundary usage
- MUST cover reactivity debugging (why updates don't fire)
- Priority anti-patterns:
  1. Destructuring props (breaks tracking)
  2. Using Array.map instead of `<For>`
  3. Treating components as functions that re-run (they don't)
  4. Using useState/useEffect/useRef (wrong API)
  5. Conditional rendering without `<Show>`
  6. Early returns in component body (runs once, return is permanent)

### Architecture & Core (solid-core-*)
- MUST explain the reactive graph model
- MUST cover the compilation step (what the compiler transforms)
- MUST provide version matrix (SolidJS versions, SolidStart versions)
- MUST explain the relationship between SolidJS and SolidStart
- MUST cover ecosystem packages (solid-router, etc.)

### Agents (solid-agents-*)
- Validation agent MUST detect React pattern contamination
- MUST check for all anti-patterns listed in solid-errors-* skills
- MUST validate correct signal access patterns
- MUST verify control flow component usage
