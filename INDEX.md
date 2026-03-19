# SolidJS Claude Skill Package — Skill Index

> Complete catalog of all 16 skills in this package.

## solid-core/ — Foundation (2 skills)

| Skill | Description |
|-------|-------------|
| [solid-core-reactivity-model](skills/source/solid-core/solid-core-reactivity-model/SKILL.md) | Fine-grained reactivity model, dependency graph, tracking contexts, ownership tree, synchronous execution |
| [solid-core-overview](skills/source/solid-core/solid-core-overview/SKILL.md) | API surface overview, version matrix (SolidJS 1.x/2.x, SolidStart 0.x/1.x), import reference, ecosystem map |

## solid-syntax/ — API Patterns (5 skills)

| Skill | Description |
|-------|-------------|
| [solid-syntax-signals](skills/source/solid-syntax/solid-syntax-signals/SKILL.md) | All reactive primitives: createSignal, createEffect, createMemo, createResource, batch, untrack, on, onMount |
| [solid-syntax-stores](skills/source/solid-syntax/solid-syntax-stores/SKILL.md) | Store system: createStore, setStore path syntax, produce, reconcile, unwrap, createMutable |
| [solid-syntax-jsx](skills/source/solid-syntax/solid-syntax-jsx/SKILL.md) | JSX compilation model, control flow (Show, For, Index, Switch/Match, Dynamic, Portal, Suspense) |
| [solid-syntax-components](skills/source/solid-syntax/solid-syntax-components/SKILL.md) | Component types, props with splitProps/mergeProps, children() helper, ref patterns, use: directives, events |
| [solid-syntax-context](skills/source/solid-syntax/solid-syntax-context/SKILL.md) | Context API: createContext, useContext, Provider pattern, reactive context, typed context, custom hooks |

## solid-impl/ — Implementation Workflows (4 skills)

| Skill | Description |
|-------|-------------|
| [solid-impl-routing](skills/source/solid-impl/solid-impl-routing/SKILL.md) | Solid Router: A component, Route config, navigation hooks, route matching, lazy loading, preloading |
| [solid-impl-solidstart](skills/source/solid-impl/solid-impl-solidstart/SKILL.md) | SolidStart: file-based routing, query/createAsync, server functions, actions, SSR/streaming, API routes |
| [solid-impl-state-patterns](skills/source/solid-impl/solid-impl-state-patterns/SKILL.md) | State architecture: signals vs stores vs context, global state, derived state, form state, composition |
| [solid-impl-testing](skills/source/solid-impl/solid-impl-testing/SKILL.md) | Testing with solid-testing-library: render, screen queries, fireEvent, async patterns, reactive state |

## solid-errors/ — Error Handling & Anti-Patterns (3 skills)

| Skill | Description |
|-------|-------------|
| [solid-errors-react-contamination](skills/source/solid-errors/solid-errors-react-contamination/SKILL.md) | Complete catalog of React anti-patterns that break SolidJS reactivity, with correct SolidJS equivalents |
| [solid-errors-reactivity-debugging](skills/source/solid-errors/solid-errors-reactivity-debugging/SKILL.md) | Diagnose reactivity issues: effects not firing, lost tracking, async tracking loss, stale closures |
| [solid-errors-error-handling](skills/source/solid-errors/solid-errors-error-handling/SKILL.md) | ErrorBoundary with reset, Suspense boundaries, nested boundaries, error recovery patterns |

## solid-agents/ — Code Review & Scaffolding (2 skills)

| Skill | Description |
|-------|-------------|
| [solid-agents-review](skills/source/solid-agents/solid-agents-review/SKILL.md) | Validates SolidJS code: signal access, control flow, props handling, store mutations, React contamination |
| [solid-agents-project-scaffolder](skills/source/solid-agents/solid-agents-project-scaffolder/SKILL.md) | Generates complete SolidJS/SolidStart project structure with Vite, TypeScript, routing, testing setup |

## Quick Navigation

- **Starting a SolidJS project?** → [solid-core-overview](skills/source/solid-core/solid-core-overview/SKILL.md), [solid-agents-project-scaffolder](skills/source/solid-agents/solid-agents-project-scaffolder/SKILL.md)
- **Converting React code?** → [solid-errors-react-contamination](skills/source/solid-errors/solid-errors-react-contamination/SKILL.md)
- **Debugging reactivity?** → [solid-errors-reactivity-debugging](skills/source/solid-errors/solid-errors-reactivity-debugging/SKILL.md)
- **Building with SolidStart?** → [solid-impl-solidstart](skills/source/solid-impl/solid-impl-solidstart/SKILL.md), [solid-impl-routing](skills/source/solid-impl/solid-impl-routing/SKILL.md)
