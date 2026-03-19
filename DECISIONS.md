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

---

### D-008: Retained All 17 Skills From Raw Masterplan
**Date:** 2026-03-19
**Decision:** All 17 skills from the raw masterplan were kept as-is during refinement. No merges, additions, or removals.
**Rationale:** Research confirmed sufficient API surface for each skill. No overlap detected — SolidJS has fewer concepts than Tauri but each concept is deeper, justifying a dedicated skill per topic.

---

### D-009: Research Organized Across Fragment Files
**Date:** 2026-03-19
**Decision:** Research references are organized across three fragment files (`reactivity-research.md`, `jsx-components-research.md`, `solidstart-ecosystem-research.md`) rather than a single vooronderzoek file.
**Rationale:** SolidJS research naturally splits into three domains. Agent prompts reference specific fragment files and sections, improving precision and reducing context overhead.

---

### D-010: React Anti-Pattern Requirement in Every Skill
**Date:** 2026-03-19
**Decision:** Every skill prompt MUST include an explicit React anti-pattern requirement. Every skill MUST show WRONG (React) vs CORRECT (SolidJS) patterns.
**Rationale:** SolidJS's core problem is React contamination. This is the package's unique value proposition — preventing Claude from generating React patterns that silently break SolidJS reactivity.

---

### D-011: solid-errors-react-contamination as Largest Skill
**Date:** 2026-03-19
**Decision:** The `solid-errors-react-contamination` skill is designated as the largest skill in the package, with the most reference file content.
**Rationale:** Research cataloged 20+ anti-patterns across reactivity and JSX/components domains. This skill is the primary defense against the core problem this package exists to solve.

---

### D-012: ErrorBoundary/Suspense Placed in errors/ Category
**Date:** 2026-03-19
**Decision:** ErrorBoundary and Suspense components are placed in `solid-errors-error-handling`, not in a syntax or impl skill.
**Rationale:** While documented in JSX/components research, these are error recovery components. Placing them in errors/ improves discoverability when developers are debugging.

---

### D-013: Testing Scoped to solid-testing-library Only
**Date:** 2026-03-19
**Decision:** The `solid-impl-testing` skill covers solid-testing-library only (render, screen, fireEvent, cleanup).
**Rationale:** No Tauri-style mock infrastructure exists for SolidJS. The testing ecosystem is centered on solid-testing-library, making broader coverage unnecessary.

---

### D-014: Topic-Specific Research Integrated in Vooronderzoek Fragments
**Date:** 2026-03-19
**Decision:** Phase 4 (deep topic research) did not produce separate `topic-research/{skill-name}-research.md` files. All topic-specific research was integrated directly into the vooronderzoek fragments in `docs/research/fragments/`.
**Rationale:** The three fragment files (`reactivity-research.md`, `jsx-components-research.md`, `solidstart-ecosystem-research.md`) already contained sufficient depth per topic. Creating additional per-skill research files would have duplicated content without adding value. This is a deliberate architectural choice, not an omission.

---

### D-015: Phases 2-6 Executed in Single Session Without Per-Phase Commits
**Date:** 2026-03-19
**Decision:** Phases 2 through 6 (research, masterplan, skill writing, validation) were executed in a single session without individual per-phase commits.
**Rationale:** The SolidJS package scope (17 skills) was small enough to complete in one session. Per-phase commits add overhead that is justified for larger packages (e.g., Blender-Bonsai with 73 skills) but unnecessary here. Future packages should evaluate scope before deciding commit granularity. Lesson learned: for packages under ~20 skills, single-session execution is efficient and viable.
