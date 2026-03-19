# SolidJS Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-19

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Kept** all 17 skills from raw masterplan | Research confirms sufficient API surface for each skill. No overlap detected — SolidJS has fewer concepts than Tauri but each concept is deeper. |
| D-02 | **Organized** research references across three fragment files | No single vooronderzoek file exists — research is split into `reactivity-research.md`, `jsx-components-research.md`, and `solidstart-ecosystem-research.md`. Agent prompts reference specific fragment files and sections. |
| D-03 | **Added** explicit React anti-pattern requirement to EVERY skill prompt | SolidJS's core problem is React contamination. Every skill must show WRONG (React) vs CORRECT (SolidJS) patterns. This is the package's unique value. |
| D-04 | **Ensured** solid-errors-react-contamination is the largest skill | Research §10 (reactivity) and §8 (jsx-components) catalog 20+ anti-patterns. This skill gets the most reference file content. |
| D-05 | **Moved** ErrorBoundary/Suspense to solid-errors-error-handling | These are error recovery components. While documented in jsx-components research, their skill home is errors/ for discoverability when debugging. |
| D-06 | **Scoped** solid-impl-testing to solid-testing-library only | No Tauri-style mock infrastructure exists for SolidJS. Testing focuses on solid-testing-library (render, screen, fireEvent, cleanup). |

**Result**: 17 raw skills → **17 definitive skills** (0 merges, 0 additions, 0 removals).

---

## Definitive Skill Inventory (17 skills)

### solid-core/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `solid-core-reactivity-model` | Fine-grained reactivity internals; reactive graph; tracking contexts; ownership model; synchronous execution; no VDOM; 2.x microtask batching | `createRoot`, `getOwner`, `runWithOwner`, tracking scopes, reactive graph | reactivity-research §1, §11 | M | None |
| `solid-core-overview` | API surface summary; version matrix (SolidJS 1.x/2.x, SolidStart 0.x/1.x, Solid Router); ecosystem map; import reference; getting started | All imports, version table, ecosystem packages | reactivity-research §Appendix, solidstart-ecosystem §11 | M | None |

### solid-syntax/ (5 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `solid-syntax-signals` | ALL reactive primitives: signals, effects, memos, resources, lifecycle, utilities | `createSignal`, `createEffect`, `createMemo`, `createResource`, `createRenderEffect`, `createComputed`, `batch`, `untrack`, `on`, `onMount`, `onCleanup`, `observable`, `from` | reactivity-research §2–§9 | L | core-reactivity-model |
| `solid-syntax-stores` | Proxy-based reactive stores; path syntax; store utilities | `createStore`, `setStore`, `produce`, `reconcile`, `unwrap`, `createMutable` | reactivity-research §6–§7 | M | core-reactivity-model |
| `solid-syntax-jsx` | JSX compilation model; control flow components; direct DOM creation vs VDOM | `Show`, `For`, `Index`, `Switch`, `Match`, `Dynamic`, `Portal`, `Suspense` | jsx-components-research §1, §5 | L | core-reactivity-model |
| `solid-syntax-components` | Component types; props handling; children helper; refs; directives; event handling | `Component`, `ParentComponent`, `VoidComponent`, `FlowComponent`, `splitProps`, `mergeProps`, `children`, `ref`, `use:` directives | jsx-components-research §2–§4, §6–§7 | L | syntax-jsx |
| `solid-syntax-context` | Context API; Provider pattern; reactive context with signals; typed context; custom hooks | `createContext`, `useContext`, Provider | solidstart-ecosystem §9 | S | syntax-signals |

### solid-impl/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `solid-impl-solidstart` | SolidStart meta-framework: file routing, data loading, server functions, actions, mutations, SSR, hydration, API routes | `query`, `createAsync`, `createAsyncStore`, `action`, `useSubmission`, `"use server"`, `redirect`, `renderToStream`, `hydrate`, FileRoutes | solidstart-ecosystem §1–§7 | L | syntax-signals, syntax-jsx |
| `solid-impl-routing` | Solid Router standalone: components, hooks, navigation, route config, lazy loading, preloading | `A`, `Route`, `Router`, `useNavigate`, `useParams`, `useSearchParams`, `useLocation`, `useBeforeLeave`, `lazy`, `usePreloadRoute` | solidstart-ecosystem §8 | M | syntax-jsx |
| `solid-impl-state-patterns` | Global state architecture: context + stores, derived state, form state, when to use signals vs stores vs context | `createContext`, `createStore`, `createSignal`, `createMemo`, `splitProps` | reactivity-research §2, §6; solidstart-ecosystem §9 | M | syntax-signals, syntax-stores, syntax-context |
| `solid-impl-testing` | Testing SolidJS: solid-testing-library, render/screen/fireEvent, cleanup, async testing, testing reactive state | `render`, `screen`, `fireEvent`, `cleanup`, `waitFor` | solidstart-ecosystem §10.2 (ecosystem mention) | M | syntax-signals, syntax-components |

### solid-errors/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `solid-errors-react-contamination` | COMPLETE React anti-pattern catalog: AP-001 through AP-020+. Every destructuring, useState, useEffect, Array.map, ternary, early return, key prop, useRef, children-as-ReactNode, re-render assumption pattern | All SolidJS APIs (corrective patterns) | reactivity-research §10; jsx-components-research §8 | L | ALL syntax skills |
| `solid-errors-reactivity-debugging` | Debugging lost reactivity: broken tracking, effects not firing, store updates not propagating, conditional signal access, async tracking loss, DevTools guidance | `createEffect`, `createSignal`, `createStore`, tracking scopes | reactivity-research §1 (tracking), §10 (anti-patterns 7-9) | M | syntax-signals, syntax-stores |
| `solid-errors-error-handling` | ErrorBoundary usage, Suspense boundaries, nested error/suspense, resetErrorBoundary, error recovery patterns | `ErrorBoundary`, `Suspense`, `createResource` | jsx-components-research §5.7–§5.8 | M | syntax-jsx |

### solid-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `solid-agents-review` | Validation checklist for SolidJS code: signal access patterns, control flow components, props handling, store mutations, event handling, React contamination detection | All validation rules from all skills | All research fragments (anti-pattern sections) | M | ALL syntax + impl + errors skills |
| `solid-agents-project-scaffolder` | Generate SolidJS/SolidStart project: Vite config, TypeScript config, component structure, routing, state management, testing setup | All scaffolding patterns | solidstart-ecosystem §1.1; reactivity-research §Appendix | L | ALL core + syntax skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-reactivity-model`, `core-overview` | 2 | None | Foundation skills, no dependencies |
| 2 | `syntax-signals`, `syntax-stores`, `syntax-jsx` | 3 | Batch 1 | Core reactive primitives + JSX model |
| 3 | `syntax-components`, `syntax-context`, `errors-react-contamination` | 3 | Batch 1–2 | Component patterns + THE anti-pattern catalog |
| 4 | `impl-solidstart`, `impl-routing`, `impl-state-patterns` | 3 | Batch 1–3 | Implementation patterns |
| 5 | `impl-testing`, `errors-reactivity-debugging`, `errors-error-handling` | 3 | Batch 1–4 | Testing + error skills |
| 6 | `agents-review`, `agents-project-scaffolder` | 2 | ALL above | Agent skills reference all other skills |

**Total**: 17 skills across 6 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package
REACTIVITY_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\docs\research\fragments\reactivity-research.md
JSX_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\docs\research\fragments\jsx-components-research.md
SOLIDSTART_RESEARCH = C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\docs\research\fragments\solidstart-ecosystem-research.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: solid-core-reactivity-model

```
## Task: Create the solid-core-reactivity-model skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-core\solid-core-reactivity-model\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for createRoot, getOwner, runWithOwner, tracking scope primitives)
3. references/examples.md (reactive graph examples, tracking demos, ownership patterns)
4. references/anti-patterns.md (React mental model violations: re-render assumption, VDOM thinking, dependency arrays)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-core-reactivity-model
description: "Guides SolidJS fine-grained reactivity model including reactive dependency graph, tracking contexts, ownership tree, synchronous execution model, and the fundamental difference from React's virtual DOM. Activates when reasoning about SolidJS reactivity, understanding why components run once, debugging tracking issues, or explaining how SolidJS updates the DOM without a virtual DOM."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Fine-grained reactivity: reactive dependency graph, automatic tracking, no VDOM
- Tracking scopes: what IS tracked (createEffect, createMemo, createRenderEffect, createComputed, JSX expressions) vs what is NOT (component body, event handlers, onMount, untrack)
- Ownership tree: createRoot, getOwner, runWithOwner, disposal semantics
- Synchronous execution model (1.x) vs microtask-batched model (2.x)
- The fundamental mental model: components run ONCE, only reactive expressions re-execute
- How JSX compilation creates direct DOM nodes with reactive bindings
- Key 2.x reactivity changes: microtask batching, flush(), async first-class

### Research Sections to Read
From reactivity-research.md:
- Section 1: Fine-Grained Reactivity Model (lines 8-49)
- Section 11: SolidJS 2.x Changes (lines 1257-1299)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples — all examples MUST be valid SolidJS
- ALWAYS show the WRONG (React mental model) vs CORRECT (SolidJS) pattern
- Include version annotations (SolidJS 1.x vs 2.x)
- Include Critical Warnings section with NEVER rules
- Include a diagram/table of tracking scopes vs non-tracking scopes
```

#### Prompt: solid-core-overview

```
## Task: Create the solid-core-overview skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-core\solid-core-overview\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete import map: solid-js, solid-js/store, solid-js/web, @solidjs/router, @solidjs/start)
3. references/examples.md (minimal app setup, SolidStart setup, version-specific patterns)
4. references/anti-patterns.md (wrong imports, version mismatches, ecosystem confusion)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-core-overview
description: "Provides SolidJS API surface overview including version matrix for SolidJS 1.x/2.x and SolidStart 0.x/1.x, complete import reference for all entry points, ecosystem package map, and getting started guidance. Activates when starting a SolidJS project, checking API availability, looking up import paths, or understanding the SolidJS ecosystem."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Version matrix: SolidJS 1.x (stable), SolidJS 2.x (beta), SolidStart 0.x (legacy), SolidStart 1.0 (stable), Solid Router versions
- Complete import map: solid-js, solid-js/store, solid-js/web, @solidjs/router, @solidjs/start
- Ecosystem packages: solid-router, solid-primitives, Kobalte, solid-testing-library, solid-devtools, babel-preset-solid
- Key version dependencies: SolidStart 1.0 requires @solidjs/router 0.15+, router requires solid-js 1.8.4+
- Getting started: npm init solid@latest, project structure, Vite config
- API categories at a glance: reactivity, stores, control flow, components, lifecycle, rendering

### Research Sections to Read
From reactivity-research.md:
- Appendix: Complete Import Map (lines 1303-1346)
- Section 11: SolidJS 2.x Changes (lines 1257-1299)

From solidstart-ecosystem-research.md:
- Section 1.1: Getting Started (lines 31-68)
- Section 10: Ecosystem (lines 991-1044)
- Section 11: SolidJS Version Matrix (lines 1048-1083)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- Include version matrix table as Quick Reference
- Include complete import map table
- Include ecosystem package table with npm names and purposes
- Include Critical Warnings about version compatibility
```

---

### Batch 2

#### Prompt: solid-syntax-signals

```
## Task: Create the solid-syntax-signals skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-syntax\solid-syntax-signals\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (ALL API signatures: createSignal, createEffect, createMemo, createResource, createRenderEffect, createComputed, batch, untrack, on, onMount, onCleanup, observable, from)
3. references/examples.md (usage patterns for each primitive, reactive composition patterns)
4. references/anti-patterns.md (React equivalents: useState, useEffect, useMemo, useRef patterns that break SolidJS)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-syntax-signals
description: "Guides ALL SolidJS reactive primitives including createSignal, createEffect, createMemo, createResource, createRenderEffect, createComputed, batch, untrack, on, onMount, onCleanup, observable, and from. Activates when creating signals, writing effects, computing derived values, fetching async data, batching updates, or managing component lifecycle in SolidJS."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **createSignal**: Accessor/Setter pattern, SignalOptions (equals, name), functional updates
- **createEffect**: Automatic tracking, execution timing (after render), cleanup via onCleanup, nested effects, previous value
- **createMemo**: Cached derived values, IS a reactive source (unlike React useMemo), equality checking
- **createResource**: Async data with loading/error/state, source-based refetching, Suspense integration, mutate/refetch
- **createRenderEffect**: Synchronous during render phase, SSR behavior, vs createEffect timing table
- **createComputed**: Before-render state synchronization
- **batch**: Deferred propagation, async breaks batching, auto-batching in effects/stores
- **untrack**: Prevent tracking, initial/default prop convention
- **on**: Explicit dependency specification, defer option
- **onMount**: Non-tracking, runs once after DOM mount
- **onCleanup**: Disposal in effects/components, LIFO order
- **observable/from**: RxJS interop, external reactive system bridges

### Research Sections to Read
From reactivity-research.md:
- Section 2: Signals (lines 53-133)
- Section 3: Effects (lines 136-321)
- Section 4: Memos (lines 325-416)
- Section 5: Resources (lines 420-561)
- Section 8: Reactive Utilities — batch, untrack, on, observable, from (lines 824-981)
- Section 9: Lifecycle — onMount, onCleanup (lines 985-1079)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples — all examples MUST be valid SolidJS
- ALWAYS show the WRONG (React) vs CORRECT (SolidJS) for each major primitive
- Include Quick Reference table of ALL primitives with one-line purpose
- Include Decision Tree: when to use signal vs memo vs effect vs resource
- Include effect execution timing table (createEffect vs createRenderEffect vs createComputed)
- Include Critical Warnings: signal getter must be called as function, effects have no dependency arrays
```

#### Prompt: solid-syntax-stores

```
## Task: Create the solid-syntax-stores skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-syntax\solid-syntax-stores\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures: createStore, setStore path syntax, produce, reconcile, unwrap, createMutable)
3. references/examples.md (store patterns: nested updates, array operations, computed getters, API sync)
4. references/anti-patterns.md (destructuring stores, spread-replace, direct mutation without createMutable)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-syntax-stores
description: "Guides SolidJS store system including createStore with proxy-based reactivity, setStore path syntax for surgical updates, produce for Immer-style mutations, reconcile for data diffing, unwrap for plain objects, and createMutable for direct mutation. Activates when managing complex state, updating nested objects, working with arrays in stores, or choosing between signals and stores."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **createStore**: Proxy-based reactivity, lazy signal creation per property, fine-grained nested tracking
- **setStore path syntax**: String paths, index access, functional updates, array operations, range updates, filter predicates, object merge behavior
- **produce**: Immer-style mutation syntax, limitations (no Sets/Maps), combination with path syntax
- **reconcile**: Data diffing for API responses, key option (default "id"), merge option
- **unwrap**: Strip reactive proxy, use cases (serialization, third-party libs)
- **createMutable**: Direct mutation, MobX/Vue-like, computed getters/setters, when to use vs createStore
- **Stores vs Signals**: Decision tree — when to use which
- **Store access syntax**: Direct property access (store.x) NOT function call

### Research Sections to Read
From reactivity-research.md:
- Section 6: Stores — createStore (lines 565-690)
- Section 7: Store Utilities — produce, reconcile, unwrap, createMutable (lines 694-820)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (React spread/replace) vs CORRECT (SolidJS path syntax) patterns
- Include Stores vs Signals comparison table
- Include setStore path syntax cheat sheet
- Include Critical Warnings: destructuring stores kills reactivity, store.x NOT store.x()
```

#### Prompt: solid-syntax-jsx

```
## Task: Create the solid-syntax-jsx skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-syntax\solid-syntax-jsx\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for Show, For, Index, Switch/Match, Dynamic, Portal, Suspense — all props and type definitions)
3. references/examples.md (control flow patterns, conditional rendering, list rendering, portals, suspense)
4. references/anti-patterns.md (Array.map, ternary, switch/case, key prop — all JSX anti-patterns)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-syntax-jsx
description: "Guides SolidJS JSX compilation model and control flow components including Show, For, Index, Switch/Match, Dynamic, Portal, and Suspense. Explains how SolidJS compiles JSX to direct DOM creation (no virtual DOM), attribute handling, namespaced attributes (attr:, prop:, class:, on:, use:, bool:), and style objects. Activates when writing SolidJS templates, rendering lists, conditional rendering, switching between views, or understanding JSX compilation."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **JSX compilation model**: Direct DOM creation, no VDOM, static vs dynamic content optimization
- **Attribute handling**: Declaration order, namespaced prefixes (attr:, prop:, class:, on:, use:, bool:, @once)
- **Style object syntax**: Double curly braces, kebab-case property names
- **Show**: when/fallback/keyed, render function children with narrowed accessor
- **For**: each/fallback, callback signature (item: T, index: () => number), keyed by reference
- **Index**: each/fallback, callback signature (item: () => T, index: number), keyed by index, For vs Index decision
- **Switch/Match**: Sequential evaluation, mutual exclusivity, fallback, render function children
- **Dynamic**: component prop (Component | string), prop forwarding, polymorphic "as" pattern
- **Portal**: mount target, useShadow, isSVG, event propagation through component tree
- **Suspense**: Resource tracking, DOM creation timing, nested Suspense, vs Show

### Research Sections to Read
From jsx-components-research.md:
- Section 1: JSX Compilation Model (lines 9-68)
- Section 5: Control Flow Components — all subsections (lines 362-825)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (Array.map/ternary/switch) vs CORRECT (For/Show/Switch) patterns
- Include For vs Index decision table
- Include control flow component Quick Reference table
- Include namespaced attribute prefix table
- Include Critical Warnings: NEVER use Array.map for lists, NEVER use ternary for complex conditionals
```

---

### Batch 3

#### Prompt: solid-syntax-components

```
## Task: Create the solid-syntax-components skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-syntax\solid-syntax-components\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Component types, splitProps, mergeProps, children() helper, ref patterns, directive API, event handler types)
3. references/examples.md (component patterns, props handling, children resolution, ref forwarding, custom directives, event binding)
4. references/anti-patterns.md (destructuring props, useRef, forwardRef, React children patterns, event handler mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-syntax-components
description: "Guides SolidJS component patterns including Component/ParentComponent/VoidComponent/FlowComponent types, props handling with splitProps and mergeProps, children() helper for resolving children, ref patterns, use: directives, and event handling with delegation. Activates when creating SolidJS components, handling props, working with children, using refs, creating directives, or handling events."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Component types**: Component, ParentComponent, VoidComponent, FlowComponent — when to use each
- **Props handling**: NEVER destructure, access props.x directly, wrap in () => props.x for derived values
- **splitProps**: Reactive destructuring alternative, multiple groups, rest props
- **mergeProps**: Default props pattern, reactivity preservation, override order
- **children() helper**: Why props.children is a getter, resolution and caching, toArray(), render props pattern
- **Refs**: Variable assignment with !: assertion, callback pattern, onMount timing, signal refs for conditional elements
- **Ref forwarding**: Parent-to-child, child receives callback regardless of parent declaration
- **Directives (use:)**: Directive signature (element, accessor), creating reusable behaviors, TypeScript declaration, cleanup
- **Event handling**: Delegated events (onClick) vs native events (on:scroll), event delegation list (23 events), array binding syntax, TypeScript event types
- **Event delegation caveats**: stopPropagation with delegated events, portal event propagation

### Research Sections to Read
From jsx-components-research.md:
- Section 2: Component Basics (lines 71-143)
- Section 3: Props Handling (lines 146-278)
- Section 4: Children (lines 282-358)
- Section 6: Event Handling (lines 828-899)
- Section 7: Refs and Directives (lines 903-1053)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (destructuring, useRef, forwardRef, React children) vs CORRECT (SolidJS) patterns
- Include Component type selection Decision Tree
- Include event delegation list table
- Include React vs SolidJS ref comparison table
- Include Critical Warnings: NEVER destructure props, NEVER access props.children multiple times without children() helper
```

#### Prompt: solid-syntax-context

```
## Task: Create the solid-syntax-context skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-syntax\solid-syntax-context\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (createContext, useContext, Provider — full type signatures)
3. references/examples.md (basic context, reactive context with signals, typed context, custom hook pattern, nested contexts)
4. references/anti-patterns.md (missing Provider, untyped context, non-reactive context values)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-syntax-context
description: "Guides SolidJS Context API including createContext, useContext, Provider pattern, reactive context with signals and stores, typed context with TypeScript, custom hook patterns, and nested context overrides. Activates when sharing state between components, creating providers, using context for dependency injection, or implementing global state patterns."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **createContext**: Default value, generic typing, context object structure
- **useContext**: Reading context value, undefined handling, error on missing Provider
- **Provider pattern**: value prop, wrapping component tree
- **Reactive context**: Passing signals and stores through context, maintaining reactivity
- **Typed context**: TypeScript generics, undefined default handling, type narrowing
- **Custom hook pattern**: useMyContext() wrapper with error checking (recommended pattern)
- **Nested contexts**: Inner Provider overrides outer for same context
- **Context + stores**: Combining createStore with createContext for global reactive state
- **Decision tree**: When context vs props vs global signals

### Research Sections to Read
From solidstart-ecosystem-research.md:
- Section 9: Context API (lines 925-987)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS include the custom hook pattern with error checking as the recommended approach
- Include Decision Tree: context vs props vs global signals
- Include Critical Warnings: ALWAYS provide typed context, NEVER use context without a Provider guard
- This is a smaller skill (S complexity) — keep it focused and concise
```

#### Prompt: solid-errors-react-contamination

```
## Task: Create the solid-errors-react-contamination skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-errors\solid-errors-react-contamination\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (anti-pattern detection rules: what code patterns to look for)
3. references/examples.md (COMPLETE catalog: every AP-NNN with WRONG React + CORRECT SolidJS side by side)
4. references/anti-patterns.md (consolidated anti-pattern quick reference table with detection + fix per pattern)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-errors-react-contamination
description: "COMPLETE catalog of React anti-patterns that break SolidJS reactivity. Covers destructured props, useState vs createSignal, useEffect vs createEffect, Array.map vs For, ternary vs Show, early returns, key prop, useRef vs ref, children as ReactNode, re-render assumption, useRouter vs useNavigate, and every other React-to-SolidJS contamination pattern. Activates when reviewing SolidJS code for React patterns, debugging reactivity breaks caused by React habits, or converting React code to SolidJS."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
Every anti-pattern numbered AP-NNN. The SKILL.md contains the Quick Reference table and top-priority patterns. The references/ files contain the COMPLETE catalog.

**Priority anti-patterns (in SKILL.md):**
- AP-001: Destructuring props (kills tracking)
- AP-002: Destructuring signals (snapshot problem)
- AP-003: useState vs createSignal (value vs getter function)
- AP-004: useEffect vs createEffect (dependency array vs auto-tracking)
- AP-005: useMemo vs createMemo (not reactive source vs IS reactive source)
- AP-006: Component re-render assumption (runs once, not every state change)
- AP-007: Conditional signal access (breaks tracking)
- AP-008: Early return before signal access (prevents tracking)
- AP-009: Storing signal value in variable (stale snapshot)
- AP-010: Spreading props (may break tracking)

**Extended catalog (in references/):**
- AP-011: Array.map vs For component
- AP-012: Ternary vs Show component
- AP-013: switch/case vs Switch/Match
- AP-014: key prop (meaningless in SolidJS)
- AP-015: useRef vs let ref / ref callback
- AP-016: React.createElement vs direct DOM
- AP-017: children as ReactNode vs children() helper
- AP-018: useEffect cleanup return vs onCleanup
- AP-019: useRouter vs useNavigate
- AP-020: Data fetching in useEffect vs createAsync/query

### Research Sections to Read
From reactivity-research.md:
- Section 10: React vs SolidJS Anti-Patterns — Reactivity (lines 1083-1254)

From jsx-components-research.md:
- Section 8: React vs SolidJS Anti-Patterns — JSX/Components (lines 1056-1253)

From solidstart-ecosystem-research.md:
- Section 12: React vs SolidJS Anti-Patterns — Routing/SSR (lines 1086-1193)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- EVERY anti-pattern MUST show: React pattern (WRONG) + SolidJS fix (CORRECT) + Detection rule + Why it breaks
- Format: AP-NNN: Name | React Pattern | Why It Fails | SolidJS Fix | Detection
- This is the MOST IMPORTANT skill in the package — it IS the reason this package exists
- Include Quick Reference table of all AP-NNN patterns
- Include detection rules that can be used for automated code review
- Include Critical Warnings section summarizing the top 5 most common contamination patterns
```

---

### Batch 4

#### Prompt: solid-impl-solidstart

```
## Task: Create the solid-impl-solidstart skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-impl\solid-impl-solidstart\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures: query, createAsync, createAsyncStore, action, useSubmission, useAction, redirect, json, reload, revalidate, renderToString, renderToStream, hydrate, HydrationScript, isServer, FileRoutes)
3. references/examples.md (file routing patterns, data loading patterns, server function patterns, action/form patterns, SSR patterns, API route patterns)
4. references/anti-patterns.md (Next.js patterns, getServerSideProps, useEffect data fetching, React Router patterns)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-impl-solidstart
description: "Guides SolidStart meta-framework including file-based routing, data loading with query and createAsync, server functions with 'use server' directive, actions and mutations with form integration, SSR and streaming with renderToStream, hydration, and API routes. Activates when building SolidStart applications, implementing server-side data loading, creating API routes, handling form submissions, or configuring SSR."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidStart 1.x with SolidJS 1.x/2.x and TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **File-based routing**: routes/ directory, nested routes, dynamic [param], optional [[param]], catch-all [...param], route groups (), nested layouts, index alternatives
- **Data loading**: query() with caching/deduplication, createAsync, createAsyncStore, route preloading
- **Server functions**: "use server" directive (function-level and file-level), serialization requirements, single-flight mutations
- **Actions**: action() wrapper, form integration (action={} method="post"), .with() for extra args, useAction for programmatic calls, useSubmission/useSubmissions tracking
- **Response helpers**: redirect (thrown, not returned), json, reload, revalidate, automatic query revalidation
- **SSR**: renderToString (sync), renderToStream (streaming), onCompleteShell/onCompleteAll callbacks
- **Hydration**: hydrate(), HydrationScript, event capture, isServer
- **API routes**: HTTP method handlers (GET/POST/PATCH/DELETE), APIEvent type, session/cookie access via vinxi/http
- **SolidStart 0.x vs 1.0 differences**: createServerData$ → query, createServerAction$ → action

### Research Sections to Read
From solidstart-ecosystem-research.md:
- Section 1: SolidStart Overview (lines 8-68)
- Section 2: File-Based Routing (lines 72-203)
- Section 3: Data Loading (lines 207-378)
- Section 4: Server Functions (lines 382-443)
- Section 5: Actions and Mutations (lines 447-557)
- Section 6: SSR and Hydration (lines 560-662)
- Section 7: API Routes (lines 665-732)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (Next.js/React Router) vs CORRECT (SolidStart) for data loading and routing
- Include file routing convention table
- Include SolidStart 0.x → 1.0 migration quick reference
- Include Critical Warnings: redirect is THROWN not returned, "use server" functions MUST be async
```

#### Prompt: solid-impl-routing

```
## Task: Create the solid-impl-routing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-impl\solid-impl-routing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures: A, Route, Router, HashRouter, MemoryRouter, Navigate, all hooks — useNavigate, useParams, useSearchParams, useLocation, useMatch, useIsRouting, useBeforeLeave, useCurrentMatches, usePreloadRoute, lazy)
3. references/examples.md (router setup, navigation patterns, route guards, lazy loading, preloading, nested routes)
4. references/anti-patterns.md (React Router patterns: Link vs A, element vs component, useRouter, loader vs preload)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-impl-routing
description: "Guides Solid Router including A component, Route configuration, Router setup, navigation hooks (useNavigate, useParams, useSearchParams, useLocation, useBeforeLeave), route matching, lazy loading with lazy(), route preloading, and config-based routing. Activates when implementing client-side routing, navigating between pages, reading URL parameters, guarding routes, or lazy-loading route components."
license: MIT
compatibility: "Designed for Claude Code. Requires @solidjs/router 0.15+ with SolidJS 1.x/2.x and TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Router setup**: Router, HashRouter, MemoryRouter
- **A component**: href, activeClass, inactiveClass, end, noScroll, replace, state
- **Route component**: path, component, preload, matchFilters, children (nested routes), multiple paths
- **Navigate component**: Programmatic redirect
- **useNavigate**: Navigate function, options (replace, scroll, state), numeric navigation (back/forward)
- **useParams**: Dynamic route parameter access
- **useSearchParams**: Query string read/write
- **useLocation**: pathname, search, hash, state
- **useMatch**: Route matching check
- **useIsRouting**: Navigation transition state
- **useBeforeLeave**: Route guard (unsaved changes warning)
- **useCurrentMatches**: All matched route segments
- **usePreloadRoute**: Trigger preloading on hover
- **lazy()**: Code splitting for route components
- **Route config objects**: Alternative to JSX route definitions

### Research Sections to Read
From solidstart-ecosystem-research.md:
- Section 8: Solid Router Standalone (lines 736-922)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (React Router: Link, element prop, useRouter) vs CORRECT (Solid Router: A, component prop, useNavigate)
- Include React Router ↔ Solid Router comparison table
- Include hook Quick Reference table
- Include Critical Warnings: NEVER use element={<Component />}, ALWAYS use component={Component}
```

#### Prompt: solid-impl-state-patterns

```
## Task: Create the solid-impl-state-patterns skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-impl\solid-impl-state-patterns\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (pattern signatures: context+store pattern, derived state patterns, form state pattern)
3. references/examples.md (global state, derived state, form management, counter provider, theme provider)
4. references/anti-patterns.md (Redux patterns, prop drilling, global mutable variables, unnecessary state)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-impl-state-patterns
description: "Guides SolidJS state management architecture including when to use signals vs stores vs context, global state with context and stores, derived state with createMemo, form state patterns, and state composition strategies. Activates when designing state architecture, choosing between signals and stores, implementing global state, managing form state, or structuring state for a SolidJS application."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Decision tree**: When to use signals vs stores vs context
  - Signals: primitive values, component-local state
  - Stores: complex/nested objects, shared mutable state
  - Context: cross-component state sharing, dependency injection
- **Global state pattern**: createContext + createStore + Provider + custom hook
- **Derived state**: createMemo for computed values, derived signals (function wrapping), avoiding unnecessary effects
- **Form state**: Controlled vs uncontrolled, store-based forms, validation patterns
- **State composition**: Multiple contexts, nested providers, state slicing with splitProps
- **External state integration**: from() for RxJS/external stores, reconcile() for syncing

### Research Sections to Read
From reactivity-research.md:
- Section 2: Signals — basics (lines 53-133)
- Section 6: Stores — basics (lines 565-690)

From solidstart-ecosystem-research.md:
- Section 9: Context API (lines 925-987)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show WRONG (Redux/useReducer/prop drilling) vs CORRECT (SolidJS context+store)
- Include state selection Decision Tree as the primary guide
- Include Critical Warnings: NEVER use createEffect to sync derived state (use createMemo), NEVER store signal values in variables
```

---

### Batch 5

#### Prompt: solid-impl-testing

```
## Task: Create the solid-impl-testing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-impl\solid-impl-testing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures: render, screen, fireEvent, cleanup, waitFor from solid-testing-library)
3. references/examples.md (component tests, signal tests, store tests, async tests, event tests)
4. references/anti-patterns.md (React Testing Library habits that don't translate, missing cleanup, wrong async patterns)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-impl-testing
description: "Guides testing SolidJS applications with solid-testing-library including render, screen queries, fireEvent, cleanup, async testing patterns, and testing reactive state. Activates when writing tests for SolidJS components, testing signal reactivity, testing store updates, or setting up test infrastructure."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript and solid-testing-library."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Setup**: Installing solid-testing-library, vitest/jest configuration, babel-preset-solid for tests
- **render**: Rendering components in test environment, container access, unmount
- **screen queries**: getByText, getByRole, queryByText, findByText — same as Testing Library conventions
- **fireEvent**: Click, input, change, submit — triggering SolidJS event handlers
- **cleanup**: ALWAYS call in afterEach, automatic with vitest/jest-dom
- **waitFor**: Async assertion for reactive updates
- **Testing signals**: Wrapping signal reads in reactive context, testing signal updates
- **Testing stores**: Store mutation assertions, nested property tracking
- **Testing async**: createResource, Suspense, error boundaries in tests
- **Test file organization**: *.test.tsx, co-location patterns

### Research Sections to Read
From solidstart-ecosystem-research.md:
- Section 10.2: Other Notable Ecosystem Packages (line 1041 — solid-testing-library mention)

Note: solid-testing-library follows Testing Library conventions. Use WebFetch to verify API at https://github.com/solidjs/solid-testing-library if needed during skill writing.

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- ALWAYS show how SolidJS testing differs from React Testing Library (component runs once, reactive updates)
- Include cleanup pattern as mandatory (Critical Warning)
- Include test setup Quick Reference
```

#### Prompt: solid-errors-reactivity-debugging

```
## Task: Create the solid-errors-reactivity-debugging skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-errors\solid-errors-reactivity-debugging\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (debugging tools: solid-devtools, console.log strategies, tracking scope identification)
3. references/examples.md (debugging scenarios: effect not firing, store not updating, lost tracking, async breaks)
4. references/anti-patterns.md (debugging mistakes: console.log in wrong scope, missing tracking context)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-errors-reactivity-debugging
description: "Diagnoses and resolves SolidJS reactivity issues including effects not firing, store updates not propagating, lost tracking from destructuring, conditional signal access breaking dependencies, async tracking loss, and stale closures. Provides debugging strategies with solid-devtools and systematic diagnostic flowcharts. Activates when SolidJS reactivity is not working, effects are not updating, store changes are not reflected in UI, or debugging reactive dependency graphs."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Effect not firing**: Signal not accessed in tracking scope, conditional access, destructured before effect
- **Store update not propagating**: Wrong setStore syntax, destructured store property, mutation without setter
- **Lost tracking from destructuring**: Props, signals, and stores — the triple threat
- **Conditional signal access**: if/early return prevents tracking, solution: access all signals first
- **Async tracking loss**: await breaks synchronous tracking, solutions
- **Stale closures**: Signal value captured in closure instead of calling getter
- **Debugging strategies**: solid-devtools browser extension, createEffect logging, tracking scope identification
- **Diagnostic flowchart**: "My UI is not updating" → systematic steps to identify cause
- **2.x differences**: Microtask batching debugging, flush() for immediate propagation

### Research Sections to Read
From reactivity-research.md:
- Section 1: Tracking scopes (lines 12-28) — what IS and IS NOT tracked
- Section 10: Anti-Patterns 7-9 (lines 1186-1254) — conditional access, early return, snapshot problem
- Section 11: 2.x changes (lines 1257-1299) — microtask batching impact on debugging

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- Format as diagnostic table: Symptom → Cause → Fix
- Include a diagnostic flowchart in SKILL.md
- Include Critical Warnings for the most common tracking breaks
```

#### Prompt: solid-errors-error-handling

```
## Task: Create the solid-errors-error-handling skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-errors\solid-errors-error-handling\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures: ErrorBoundary props, Suspense props, SuspenseList, catchError)
3. references/examples.md (ErrorBoundary patterns, Suspense patterns, nested boundaries, error recovery, Suspense + ErrorBoundary composition)
4. references/anti-patterns.md (try/catch in components, missing boundaries, error in fallback)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-errors-error-handling
description: "Guides SolidJS error handling including ErrorBoundary usage with reset capability, Suspense boundaries for loading states, nested error and suspense boundaries, what errors ARE and ARE NOT caught, and error recovery patterns. Activates when implementing error boundaries, handling loading states with Suspense, recovering from errors, or combining ErrorBoundary with Suspense."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **ErrorBoundary**: fallback as JSX or function (err, reset), what it catches (rendering, effects, memos, resources), what it does NOT catch (event handlers, setTimeout, outside reactive scope)
- **Suspense**: fallback, resource tracking, DOM creation timing (created but not attached), nested Suspense (independent resolution)
- **ErrorBoundary + Suspense composition**: Wrapping order, error in resource vs render
- **Nested boundaries**: Inner catches first, error in fallback propagates to parent
- **Error recovery**: reset function, refetch resources, state reset patterns
- **Suspense vs Show**: Suspense creates DOM immediately (preserves state), Show destroys/recreates
- **SuspenseList**: Coordinating multiple Suspense boundaries (if available)
- **Server-side**: createEffect never runs during SSR, Suspense streaming behavior

### Research Sections to Read
From jsx-components-research.md:
- Section 5.7: ErrorBoundary (lines 699-767)
- Section 5.8: Suspense (lines 769-825)

From reactivity-research.md:
- Section 5: Resources — Suspense integration (lines 526-536)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples
- Include "What ErrorBoundary catches" vs "What it does NOT catch" table
- Include ErrorBoundary + Suspense composition pattern as Essential Pattern
- Include Critical Warnings: errors in fallback propagate to parent, NEVER rely on try/catch in component body for rendering errors
```

---

### Batch 6

#### Prompt: solid-agents-review

```
## Task: Create the solid-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-agents\solid-agents-review\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete validation checklist organized by area)
3. references/examples.md (review scenarios: good code, bad code, fixes — organized by check category)
4. references/anti-patterns.md (all React contamination patterns consolidated, organized for quick scanning)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-agents-review
description: "Validates generated SolidJS code for correctness by checking signal access patterns, control flow component usage, props handling, store mutations, event handling, context usage, and React contamination detection. Run this checklist on every SolidJS code generation to catch silent reactivity breaks. Activates when reviewing SolidJS code, validating a SolidJS project, or checking for React pattern contamination before deployment."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
Structure as a checklist that can be run against any SolidJS codebase:

**Signal Access Checks:**
- Signals called as functions in JSX: count() not count
- No signal values stored in variables outside reactive scope
- No destructured signal getters
- createMemo used for derived values, not createEffect + setSignal

**Props Checks:**
- No destructured props in function signature or body
- splitProps used when splitting props
- mergeProps used for default values
- children() helper used when accessing props.children multiple times

**Control Flow Checks:**
- For component used for lists, NOT Array.map
- Show component used for conditionals, NOT ternary (complex cases)
- Switch/Match used for multi-branch, NOT switch/case in component body
- No key prop on For/Index children

**Store Checks:**
- setStore path syntax used, NOT spread/replace
- No destructured store properties in tracking scopes
- produce used for complex mutations
- reconcile used for external data sync

**Component Checks:**
- Components return JSX, no early returns for conditional rendering
- Event handlers use correct binding (onClick, on:scroll)
- Refs use let ref!: type pattern, not useRef
- Directives declared in TypeScript module augmentation

**React Contamination Scan:**
- No useState, useEffect, useMemo, useRef, useCallback imports
- No dependency arrays in effect-like code
- No forwardRef usage
- No key prop on list items
- No React.createElement or React.Fragment

### Research Sections to Read
From ALL research fragments — consolidate anti-patterns:
- reactivity-research.md §10 (lines 1083-1254)
- jsx-components-research.md §8 (lines 1056-1253)
- solidstart-ecosystem-research.md §12 (lines 1086-1193)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in examples (except as WRONG examples)
- Structure as actionable checklist — each item: what to check, expected correct state, common failure
- Include severity levels: CRITICAL (breaks reactivity), WARNING (suboptimal), INFO (style)
- This skill is an AGENT — it should be usable as a systematic review procedure
```

#### Prompt: solid-agents-project-scaffolder

```
## Task: Create the solid-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\SolidJS-Claude-Skill-Package\skills\source\solid-agents\solid-agents-project-scaffolder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (scaffolding templates: file listings, config templates, component templates)
3. references/examples.md (complete SolidJS project scaffold, complete SolidStart project scaffold)
4. references/anti-patterns.md (scaffolding mistakes: wrong config, missing babel preset, React boilerplate)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: solid-agents-project-scaffolder
description: "Generates complete SolidJS or SolidStart project structure including Vite configuration, TypeScript configuration, component directory structure, routing setup, state management patterns, and testing setup. Produces production-ready project scaffolds with correct SolidJS idioms. Activates when creating a new SolidJS project, scaffolding a SolidStart application, setting up project infrastructure, or generating a complete project template."
license: MIT
compatibility: "Designed for Claude Code. Requires SolidJS 1.x/2.x with TypeScript and Vite."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- **Plain SolidJS scaffold**: Vite + SolidJS + TypeScript
  - vite.config.ts (vite-plugin-solid)
  - tsconfig.json (jsx: "preserve", jsxImportSource: "solid-js")
  - package.json (dependencies: solid-js, devDependencies: vite-plugin-solid, typescript)
  - src/index.tsx (render entry point)
  - src/App.tsx (root component)
  - src/components/ directory structure

- **SolidStart scaffold**: Full meta-framework
  - app.config.ts (SolidStart config)
  - src/app.tsx (root with Router + FileRoutes + Suspense)
  - src/entry-client.tsx, src/entry-server.tsx
  - src/routes/ (index.tsx, about.tsx, layout examples)
  - src/components/ (shared components)
  - src/lib/ (utilities, server functions)

- **Configuration templates**:
  - vite.config.ts for both plain and SolidStart
  - tsconfig.json with SolidJS-specific settings
  - .gitignore (node_modules, dist, .vinxi)

- **State management setup**: Context + Store provider template
- **Routing setup**: Router with example routes
- **Testing setup**: vitest.config.ts, test utils, example test file

- **Decision tree**: Plain SolidJS vs SolidStart — when to use which

### Research Sections to Read
From solidstart-ecosystem-research.md:
- Section 1.1: Getting Started — project structure (lines 31-68)
- Section 10: Ecosystem — key packages to include (lines 991-1044)

From reactivity-research.md:
- Appendix: Complete Import Map (lines 1303-1346) — correct imports for templates

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TypeScript + TSX (D-006)
- No React patterns in generated templates
- Generated code MUST follow all SolidJS idioms from other skills
- Include complete file listings with content in references/
- Include Plain SolidJS vs SolidStart decision tree
- Include Critical Warnings: NEVER use React boilerplate, ALWAYS use vite-plugin-solid, ALWAYS set jsxImportSource: "solid-js"
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── solid-core/
│   ├── solid-core-reactivity-model/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   └── solid-core-overview/
│       ├── SKILL.md
│       └── references/
│           ├── methods.md
│           ├── examples.md
│           └── anti-patterns.md
├── solid-syntax/
│   ├── solid-syntax-signals/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-syntax-stores/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-syntax-jsx/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-syntax-components/
│   │   ├── SKILL.md
│   │   └── references/
│   └── solid-syntax-context/
│       ├── SKILL.md
│       └── references/
├── solid-impl/
│   ├── solid-impl-solidstart/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-impl-routing/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-impl-state-patterns/
│   │   ├── SKILL.md
│   │   └── references/
│   └── solid-impl-testing/
│       ├── SKILL.md
│       └── references/
├── solid-errors/
│   ├── solid-errors-react-contamination/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── solid-errors-reactivity-debugging/
│   │   ├── SKILL.md
│   │   └── references/
│   └── solid-errors-error-handling/
│       ├── SKILL.md
│       └── references/
└── solid-agents/
    ├── solid-agents-review/
    │   ├── SKILL.md
    │   └── references/
    └── solid-agents-project-scaffolder/
        ├── SKILL.md
        └── references/
```
