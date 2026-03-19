# SolidJS Claude Skill Package

**Deterministic Claude AI skills for SolidJS reactive framework development.**

Built on the [Agent Skills](https://agentskills.org) open standard.

## Why This Exists

SolidJS looks like React but works fundamentally differently. Claude systematically generates React patterns that **silently fail** in SolidJS:

```tsx
// WRONG - React pattern, breaks SolidJS reactivity
const [count, setCount] = useState(0);

// CORRECT - SolidJS signal
const [count, setCount] = createSignal(0);
```

```tsx
// WRONG - Destructuring props breaks reactivity tracking
function Counter({ count, onClick }) {

// CORRECT - Access props directly to preserve reactivity
function Counter(props) {
  return <button onClick={props.onClick}>{props.count()}</button>
}
```

```tsx
// WRONG - Array.map loses reactivity, no keyed diffing
{items.map(item => <Item item={item} />)}

// CORRECT - <For> component with fine-grained updates
<For each={items()}>{(item) => <Item item={item} />}</For>
```

**AI failure rate without skills: STRUCTURAL** — React contamination is systematic, not occasional.

## Package Contents

| Category | Directory | Purpose | Skills |
|----------|-----------|---------|--------|
| Syntax | `solid-syntax/` | Signals, stores, reactivity primitives, JSX compilation | 0 |
| Implementation | `solid-impl/` | Component patterns, routing, SolidStart workflows | 0 |
| Errors | `solid-errors/` | React contamination detection, reactivity debugging | 0 |
| Core | `solid-core/` | Architecture, fine-grained reactivity model, version matrix | 0 |
| Agents | `solid-agents/` | Validation, React-pattern detection | 0 |
| **Total** | | | **0** |

## Technology Coverage

| Technology | Versions | Languages |
|-----------|----------|-----------|
| SolidJS | 1.x, 2.x | TypeScript, TSX |
| SolidStart | 0.x, 1.x | TypeScript, TSX |

## Current Progress

**Phase 1: Infrastructure & Research Setup** — 50%

See [ROADMAP.md](ROADMAP.md) for detailed progress.

## Documentation

| File | Purpose |
|------|---------|
| [CLAUDE.md](CLAUDE.md) | Protocols and session instructions |
| [ROADMAP.md](ROADMAP.md) | Project status and next steps |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Quality guarantees |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions |
| [SOURCES.md](SOURCES.md) | Official reference URLs |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Used In

- [Open PDF Studio](https://github.com/OpenAEC-Foundation/open-pdf-studio) — Tauri 2 desktop PDF editor

## Related Skill Packages

- [Tauri 2](https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package) — Desktop/mobile application framework
- [ERPNext](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) — ERP system
- [Blender-Bonsai](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) — 3D/BIM modeling

## License

MIT — see [LICENSE](LICENSE)

---

Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.
