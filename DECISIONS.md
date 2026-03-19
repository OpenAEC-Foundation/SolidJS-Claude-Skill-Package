# SolidJS Claude Skill Package — Decisions

> Architectural decisions. Numbered D-XXX. Immutable once recorded.

---

### D-001: Single Technology Package
**Date:** 2026-03-19
**Decision:** This package covers SolidJS only. No other frameworks.
**Rationale:** Focused packages produce higher-quality skills. SolidJS has unique characteristics (fine-grained reactivity, no VDOM) that require dedicated attention.

---

### D-002: 7-Phase Research-First Methodology
**Date:** 2026-03-19
**Decision:** Follow the 7-phase methodology proven in ERPNext, Blender, and Tauri packages.
**Rationale:** Research-first approach ensures skills are accurate, complete, and verified against official sources before writing begins.

---

### D-003: English-Only Content
**Date:** 2026-03-19
**Decision:** All skill content, code comments, and documentation MUST be in English.
**Rationale:** Skills are consumed by Claude globally. English ensures maximum reach. Project management files (ROADMAP etc.) may reference Dutch terms where conventional (e.g., "vooronderzoek").

---

### D-004: SolidJS 1.x and 2.x Coverage
**Date:** 2026-03-19
**Decision:** Skills MUST cover SolidJS 1.x (current stable) and note 2.x changes where applicable.
**Rationale:** SolidJS 1.x is the production version. SolidJS 2.x introduces changes to the reactivity system. Skills must serve both audiences.

---

### D-005: MIT License
**Date:** 2026-03-19
**Decision:** Repository licensed under MIT.
**Rationale:** Consistent with OpenAEC Foundation's open-source strategy and all other skill packages.

---

### D-006: TypeScript + TSX Only
**Date:** 2026-03-19
**Decision:** All code examples MUST use TypeScript with TSX. No plain JavaScript examples.
**Rationale:** TypeScript is the recommended way to use SolidJS. Plain JS examples would encourage weaker type safety. TSX is the standard template syntax for SolidJS components.

---

### D-007: SKILL.md Line Limit 500
**Date:** 2026-03-19
**Decision:** Every SKILL.md file MUST be under 500 lines. Heavy content goes in references/ subdirectory.
**Rationale:** Skills must be concise and scannable. Long files reduce Claude's ability to quickly find relevant patterns. References provide depth without bloating the main file.
