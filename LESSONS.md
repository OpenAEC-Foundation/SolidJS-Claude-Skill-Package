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

---

### L-007: YAML Frontmatter Standard — Folded Block Scalar
**Date:** 2026-03-19
**Phase:** 7 (Compliance Audit)
**Lesson:** ALWAYS use folded block scalar (`>`) for YAML description fields, NEVER quoted strings. The folded scalar avoids quoting issues and enforces the multi-line "Use when / Prevents / Covers / Keywords" structure that optimizes skill trigger matching.
**Impact:** All skill YAML frontmatter descriptions must use the `>` scalar format. This should be added to WAY_OF_WORK.md as an explicit formatting rule.

---

### L-008: Per-Phase Git Commits Are Mandatory
**Date:** 2026-03-19
**Phase:** 7 (Compliance Audit)
**Lesson:** Each methodology phase MUST have its own git commit(s). Combining all phases into a single mega-commit makes it impossible to audit phase boundaries, roll back individual phases, or verify that the workflow was followed sequentially. This was identified as ISSUE-004 in the compliance audit.
**Impact:** Future skill packages must enforce per-phase commits as a hard workflow rule. The commit message should reference the phase number (e.g., `Phase 3.1: create masterplan`).

---

### L-009: Compliance Audit as Final Quality Gate
**Date:** 2026-03-19
**Phase:** 7 (Compliance Audit)
**Lesson:** Running a methodology compliance audit after Phase 7 caught 10 issues (71% score) that were invisible during development. The audit-remediation cycle raised compliance to 90%+. ALWAYS run the audit before declaring a package complete.
**Impact:** A compliance audit step should be formalized as Phase 7.5 (or integrated into Phase 7) in the Skill-Package-Workflow-Template. No package should be tagged as v1.0.0 without passing the audit at 90%+ compliance.
