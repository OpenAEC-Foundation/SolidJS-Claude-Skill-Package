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
