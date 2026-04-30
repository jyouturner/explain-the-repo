# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.3.0] — 2026-04-30

### Added
- **Step 0 — design pass.** New `references/design-pass.md` with sibling rules (cadence, failure, zoom, topology, lifecycle siblings) and decision logic for when one diagram suffices vs when N siblings are needed (cap at 5).
- **Design-panel critique** for set-level review when N > 1: `references/design-panel-prompt.md` with a single-panelist rubric (`missing-aspect`, `wrong-cadence-split`, `unanchored-zoom`, `redundant-siblings`, `over-fanout`, `under-fanout`, `headline-unclear`).
- **Step 6 — panel critique + syntax lint + bounded revision** as a default step. Step 6 spawns three parallel subagents per diagram: two panel-critique subagents (`references/panel-prompt.md`, four-panelist semantic review with disjoint rubrics, votes unioned across the two runs) and one syntax-lint subagent (`references/syntax-lint-prompt.md`, parser-footgun scanner with auto-fixes).
- **Multi-diagram output format** for sets where N > 1 (design summary + design-panel summary + per-diagram sections + closing notes).
- **Opt-out path:** the skill now recognizes user signals like "quick diagram", "no panel", "fast", "skip the design phase" — drops to N=1, skips the panel critique, but always runs the syntax linter.
- `examples/real-repos/` — three worked examples on real codebases (`autoresearch.md` as a multi-diagram set, `recs-two-tower.md` and `graphql-node.md` as single-diagram sets with their design-pass rationale annotated).
- `design/refinement-loop.md` and `design/integrated-flow-sketch.md` — design rationale, calibration findings, and resolution notes for the open questions phase C raised.
- `evals/evals.json` updated and expanded to 7 prompts: existing prompts now expect a step-0 design summary and a step-6 panel summary; new evals exercise multi-diagram output (`multi-cadence-agent`) and opt-out (`explicit-opt-out`).

### Changed
- `SKILL.md` restructured around step 0 (design pass) + steps 1–6 (per diagram). Step 1's wording narrowed: archetype is fixed by step 0; step 1 confirms the specific job within that archetype. Step 5 (iteration-mode redraw) now routes back to step 0, not step 1, so a sprawling existing diagram can be redrawn as a multi-diagram set when warranted.
- `references/mermaid-patterns.md` § Syntax footguns: expanded footgun #6 with a concrete bracket example after a real bug shipped (`norm -.->|products[]| resolver` failed on GitHub); pre-ship checklist now lists `[]`, `()`, `{}` explicitly.
- README reflects the two-phase procedure (design + per-diagram) and lists the new reference files.
- `examples/walkthrough.md` updated with a step-0 section and a step-6 panel-summary section.

### Resolved (open questions from phase C iteration)
- Subagent vs same-context for the panel: subagent (load-bearing for avoiding self-grading bias).
- Opt-out flag: no env flag; user signals in the prompt instead.
- Archetype tie-break across two parallel runs: prefer SME-bearing run, else first run.
- Revise contract: regenerate from step 2 with adjusted plan; never edit Mermaid in place.
- Iteration-mode-redraw under phase C: route back to step 0; the original is often a single-diagram-overreach symptom.

## [0.2.0] — 2026-04-26

### Added
- `LICENSE` file (MIT) — was claimed in README but missing from the tree.
- `CONTRIBUTING.md` with guidance on what kinds of issues and PRs are most useful.
- `CHANGELOG.md` (this file).
- `examples/walkthrough.md` — a worked example applying the procedure to a multi-tenant RAG service, including the plan-then-source workflow.
- `examples/anti-patterns.md` — a catalog of failure modes commonly seen in AI-generated system diagrams, with concrete descriptions of how each one manifests.
- `evals/evals.json` — small set of test prompts exercising different diagram archetypes (request trace, batch pipeline, iteration redraw, out-of-scope ERD, ML pipeline).
- README sections: "Before / after," "Try it," "Limitations," "Rendering caveats," "Evaluation."

### Changed
- README restructured to lead with a concrete before/after comparison.
- README is now explicit about scope (C4 container level, request-trace bias) and renderer dependencies (elk vs dagre).
- Repo identifier normalized: `system-diagram-in-mermaid` (hyphens) used consistently in install paths and file references.

### Notes
- No changes to `SKILL.md` or `references/` content. The skill itself is unchanged from v0.1; this release is documentation, examples, and project hygiene.

## [0.1.0] — initial commit

### Added
- `SKILL.md` with the five-step procedure.
- `references/diagnostic-checklist.md` — failure modes for diagnosing existing diagrams.
- `references/mermaid-patterns.md` — Mermaid idioms for system diagrams.
- Initial README with install instructions and trigger examples.
