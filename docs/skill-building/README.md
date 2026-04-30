# Building Claude Code skills — three documents

Three artifacts capturing what we learned building `explain-the-repo` (formerly `system-diagram-in-mermaid`). The skill itself was non-trivial — multi-level procedure, multi-subagent critique loop, stochasticity handling, pivot from diagram-only to architecture-doc-with-diagrams — and the journey produced lessons that generalize to other skill-building work. None of this is a substitute for [Claude Code's official skill documentation](https://docs.claude.com/en/docs/claude-code/skills); it complements that with specific gotchas and a worked example.

Pick the doc that matches what you need:

- **[`checklist.md`](checklist.md)** — *one-page operational guardrails.* "Before you ship, did you…?" Designed to paste at the top of a skill-building session or post in a wiki. Each item is one question, one sentence on why it matters. Read this if you're about to build a skill and want to avoid the most common failure modes we hit.

- **[`lessons.md`](lessons.md)** — *seven insights from the journey, with the specific incidents that produced them.* More narrative than actionable; explains *why* the checklist items exist. Read this if you want context — "what does 'design the opt-out alongside the second critique pass' actually mean in practice?"

- **[`tutorial.md`](tutorial.md)** — *worked-example walkthrough of the 17 commits.* Each phase: what we did, why, what surfaced, what got committed. Read this if you're learning the methodology by example or want to understand the rationale behind a specific design choice in this skill.

## When this depth is justified (and when it isn't)

The journey we documented is unusually iterative. Most skills don't (and shouldn't) earn 17 commits with calibration runs and mid-stream pivots. The depth here was justified because the panel-critique mechanic was genuinely novel and needed empirical validation; for a skill that's "format dates as ISO-8601," half this rigor is process anxiety in disguise.

Use the checklist as a *filter*, not a *requirement*. If your skill is small, simple, and well-understood, most items are obvious or N/A. The checklist exists to catch the items that aren't obvious — the ones that bit us during this journey.
