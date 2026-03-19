# SolidJS Claude Skill Package — Lessons Learned

> Numbered lessons (L-XXX) discovered during development.

---

### L-001: React Contamination is Structural, Not Occasional
**Date:** 2026-03-19
**Phase:** 1
**Lesson:** Claude's training data is dominated by React patterns. When asked to write SolidJS code, Claude systematically produces React idioms (useState, useEffect, destructured props, Array.map) that silently fail in SolidJS. This is not an occasional mistake — it is a structural bias that requires explicit counter-training via skills.
**Impact:** Every skill must include React anti-pattern / SolidJS correction pairs. The errors/ category is critical for this package.

---

### L-002: SolidJS Components Run Once
**Date:** 2026-03-19
**Phase:** 1
**Lesson:** The most fundamental difference between React and SolidJS: SolidJS component functions execute exactly once. There is no "re-render". Reactivity happens at the signal/effect level, not the component level. This mental model difference is the root cause of most React contamination bugs.
**Impact:** This concept must be reinforced in every skill that touches components, signals, or effects.

---

### L-003: Skill Prefix Decision — solid- not solidjs-
**Date:** 2026-03-19
**Phase:** 1
**Lesson:** Chose `solid-` as the skill prefix rather than `solidjs-` for brevity. The `solidjs-` prefix would be redundant since the package name already identifies the technology. Shorter prefixes improve readability in skill lists.
**Impact:** All skill directories use `solid-{category}-{topic}` naming.

---

### L-004: Research-First Methodology Works
**Date:** 2026-03-19
**Phase:** 5
**Lesson:** Producing 3838 lines of verified research before writing any skills enabled deterministic, high-quality skill authoring. Every skill could reference concrete API details, version differences, and anti-patterns from the vooronderzoek rather than relying on model knowledge alone. The research investment paid for itself by eliminating rework.
**Impact:** Future skill packages should always complete thorough research phases before skill writing begins.

---

### L-005: Batch Delegation is Effective
**Date:** 2026-03-19
**Phase:** 5
**Lesson:** 16 skills across 6 batches were written in a single session using parallel agent execution. The 3-agents-per-batch strategy with separated file scopes prevented conflicts and allowed efficient throughput. Each agent received complete context from core files, enabling autonomous high-quality output.
**Impact:** Batch size of 3 agents with non-overlapping file scopes is the proven optimal configuration for Claude Code Agent tool.

---

### L-006: The Anti-Pattern Catalog is the Core Value
**Date:** 2026-03-19
**Phase:** 6
**Lesson:** solid-errors-react-contamination is the most important skill in the package. It directly addresses Claude's structural bias toward React patterns — the core problem this entire package exists to solve. The 31 cataloged anti-patterns (AP-001 through AP-031) with explicit wrong/right code pairs provide the highest-impact training signal for Claude.
**Impact:** When prioritizing maintenance or updates, the errors/ category skills should always be updated first.
