# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.5.0] — 2026-05-02

Driven by an end-to-end dogfood pass on a real codebase (a contract-review document AI pipeline). The dogfood surfaced nine concrete improvements; this release implements all of them. Theme: the v0.4 procedure produces docs that look right but doesn't measure whether they *work* for the stranger they're written for, and lets visual decisions drift across diagrams within a single doc. v0.5 adds a measurement layer (the reader test) and a consistency layer (the visual vocabulary lock-in), plus several smaller corrections.

### Added

- **Phase D — Reader test (`references/reader-test-prompt.md`).** A new self-contained prompt for a fresh subagent that reads the produced doc cold (no source-code access, no prior context) and answers five archetypal onboarding questions: where to start, what does load-bearing component X do (and not do), how does data flow from input to output, what persists, what's out of scope. The verdict drives a bounded one-round revision when the doc fails to onboard. Runs unconditionally — even when in-session authorship is detected (the reader test's protection is intrinsic, not contingent on session context). The eval-driven analog of the panel critiques: panels check shape; the reader test checks whether a stranger can use it.
- **Phase 0a — In-session-author detection (`SKILL.md`).** Heuristic-based detection of when the current Claude session has authored the system being documented (recent edits to the source tree, long task chain in this session, recent git commits by Claude). When triggered, the panel critiques (doc-level and per-diagram) are skipped because in-context self-grading is unreliable, and the generation notes emit a required disclosure block directing the user to a fresh session for full panel coverage. Reader test and syntax linter still run.
- **Phase 0b — Existing-docs scan (`SKILL.md`).** Before the doc design pass, scan the target repo for `README.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, `docs/*.md`, and substantive in-tree `*.md` files. Pass the summaries to Phase A so the doc plan can position the new doc complementarily, not redundantly. When overlap is high, surface a question to the user (complementary positioning / replace / merge) instead of producing a duplicate.
- **Visual vocabulary lock-in (`references/doc-design-pass.md`).** New mandatory step in Phase A: lock a 4–6 color palette and 2–4 shape vocabulary at design-pass time, with stable axis assignments that hold across every diagram in the doc. Pass the locked vocabulary as input to each per-section diagram-procedure invocation. Diagrams must declare conformance and surface any need to extend the vocabulary. Prevents the failure mode where each diagram picks its own colors and the reader has to re-learn a mapping per section.
- **Component-summary required template (`references/doc-design-pass.md`).** Explicit four-line shape — responsibility paragraph, `**Surface**:` line, `**Boundary**:` line, `**Key files**:` line — replacing the previous one-sentence guidance. The `Boundary` line is the highest-leverage part because it forces articulating negative space; flagged specifically by the doc panel when missing. Skip only when components are heterogeneous enough that uniformity would mislead, with the deviation called out in the section header.
- **Test-and-eval-surface section type (`references/doc-design-pass.md`).** New section in the taxonomy (#9, before glossary). Names where the eval lives, how to run it, what the metrics mean, what passes vs fails. Trigger signals: a `data/eval/`, `evals/`, `benchmarks/`, or similar directory; an `eval.py` / `benchmark.py` entry point; a metric prominently named in the headline. New decision rule (#6 in the rules list). Doc-panel anchor `eval-surface-buried` flags when the repo has an obvious eval surface but the doc has no section for it.
- **Lifecycle / state-and-persistence merge rule (`references/doc-design-pass.md`).** Decision rule for when to merge the two sections vs keep them separate: merge when the build-time output of one stage is the run-time input of another AND the persistence topology is the lifecycle diagram with different framing. Doc-panel anchor `lifecycle-state-overlap-not-merged` fires when both sections are present and overlap ≥ 70%.
- **GitHub README survival kit (`references/mermaid-patterns.md`).** New section "Publishing to GitHub README" with dagre-specific constraints: prefer simple shapes over hexagons, cap subgraphs at 2–3, no `direction` overrides inside subgraphs, ≤ 3 lines per node label, ≤ 15-character edge labels, 6–12 nodes is the sweet spot. Two named anti-patterns (multi-line edge labels, hexagon clusters). Replaces the previous one-line mention with actionable guidance for diagrams bound for GitHub.
- **Phase A token-budget output (`SKILL.md`).** After the doc plan is finalized but before generation begins, emit a per-phase token estimate. The user gets a "full path / light path / cancel" choice with concrete numbers attached. Light path skips Phase B/C panels but keeps the syntax linter and reader test.
- **`inconsistent-vocabulary` and `component-summary-template-violation` anchors (`references/doc-panel-prompt.md`).** Both revision-worthy. Plus `eval-surface-buried` and `lifecycle-state-overlap-not-merged` as called out above. The filter table is updated.

### Changed

- `references/diagram-procedure.md` step 2 (plain-text plan) now has five parts instead of four: a new "Vocabulary conformance" item that requires diagrams to declare which colors and shapes from the doc's locked vocabulary they will use. When the diagram procedure is invoked standalone (not from the doc skill), this part is skipped.
- `references/diagram-procedure.md` step 4 cross-links to `references/mermaid-patterns.md` § "Publishing to GitHub README" when the diagram is bound for a dagre-rendered context. Hexagon, 4+ subgraphs, multi-line edge labels, and 15+ nodes are now flagged as common dagre failure cases at design time, not after the diagram ships.
- `SKILL.md` procedure renamed Phase D (output) to Phase E. The new Phase D is the reader test. Generation-notes template formalized and now required to include a Phase 0a disclosure block when in-session authorship was detected.
- `SKILL.md`'s "Why this skill exists" updated to mention the visual-vocabulary discipline and the reader test as the two distinguishing properties of the v0.5 procedure.

### Notes

- The v0.5 changes are additive to the v0.4 procedure. Existing invocations that don't trigger any of the new conditions (small docs without eval surfaces, single-diagram docs with no in-session authorship) run essentially the same path as before, plus the reader test.
- The reader test cannot be skipped by the in-session-author detection. The detection skips the panel critiques because they're vulnerable to context bias; the reader test is designed to be intrinsically independent (no source-code access, no prior context), so the bias concern doesn't apply.
- Open question: how the in-session-author heuristic performs in practice. The signals (session-edited files, recent git activity, conversation length) are noisy. False negatives (the agent authored the system but the heuristic missed it) lead to self-graded panels; false positives lead to skipped critiques on docs that didn't need skipping. The "ask the user when uncertain" fallback is the relief valve. Empirical only.
- Open question: whether the reader test's 5-question template generalizes. The five questions are derived from the section taxonomy (where-to-start, components, architecture overview, persistence, out-of-scope) — if the taxonomy evolves, the questions may need to evolve with it.

## [0.4.0] — 2026-04-30

### Renamed
- **Skill: `system_diagram_in_mermaid` → `explain_the_repo`.** The work of v0.1–v0.3 made clear that users asking for a system diagram of a repo were actually asking for an *architecture document* containing diagrams, not diagrams alone. The diagram-set work, the multi-section format, the design-pass / panel-critique structure — all of it is shaped by the architecture-doc use case. v0.4 commits to that positioning. GitHub repo also renamed to `explain-the-repo` after the v0.4.0 commit.
- **Skill description updated** to trigger on requests like "explain this repo," "architecture of [project]," "document this service," "onboard a new engineer to [codebase]" — in addition to the diagram-only triggers from earlier versions.

### Added
- **Outer doc-level skill (`SKILL.md`).** Two-level procedure. Phase A is a doc design pass that decides what *sections* the doc needs (headline, where-to-start, architecture overview, component summaries, lifecycle, data flow, state and persistence, glossary, out-of-scope). Phase B generates each section — prose sections get written and grounded in cited files; diagram-set sections invoke the inner diagram procedure. Phase C is a doc-level coherence review for docs with more than 3 sections.
- **`references/diagram-procedure.md`** — the inner diagram procedure (steps 0–6 from the previous version's SKILL.md), now used by Phase B for diagram-set sections and usable standalone for diagram-only requests.
- **`references/doc-design-pass.md`** — section taxonomy and decision rules for Phase A. Default doc shape is 3 sections (headline + architecture overview + where-to-start); add sections per documented rules; cap at 8–10.
- **`references/doc-panel-prompt.md`** — doc-level coherence critique. Anchors include `missing-section`, `wrong-section-order`, `prose-diagram-mismatch`, `unanchored-pointers`, `not-grounded`, `where-to-start-missing-or-thin`, `no-out-of-scope-section`, `headline-overreaches`. Filter table partitions into revision-worthy and borderline.
- **Doc-level opt-out:** signals like "quick overview", "no critique", "rough sketch", "skip the design pass" drop to a minimum 3-section shape and skip the doc-panel. Inner diagram procedure's syntax linter still runs.
- **`examples/real-repos/` regenerated as full architecture docs.** The three real-repo examples (`autoresearch.md`, `recs-two-tower.md`, `graphql-node.md`) are now demonstrations of full architecture-doc output, not just diagrams. The autoresearch case shows a 5-section doc with a 4-diagram architecture overview; recs_two_tower shows a 5-section doc including a Lifecycle section; graphql-node shows a minimal 4-section doc with a single-diagram overview.

### Renamed (file moves)
- `references/design-pass.md` → `references/diagram-set-design-pass.md` (clearer distinction from the new doc-level design pass).
- `references/design-panel-prompt.md` → `references/diagram-set-panel-prompt.md`.

### Changed
- `examples/walkthrough.md` reframed as a walkthrough of the **inner diagram procedure** (the steps inside `references/diagram-procedure.md`). The full doc-level walks now live in `examples/real-repos/`.
- `evals/evals.json` bumped to 0.4.0, expanded to 8 prompts at two levels (doc-level and diagram-only). New evals: `explain-the-repo-architecture-doc` (multi-diagram doc), `explain-the-repo-simple-api` (minimal doc), `explain-the-repo-ml-system` (Lifecycle section warranted), `diagram-only-multi-cadence-agent` (inner-procedure-only multi-diagram), `explicit-opt-out` (quick overview path).
- README rewritten for the new positioning; lineage section added documenting the rename.

### Notes
- The inner diagram procedure (everything in `references/diagram-procedure.md`) is unchanged from v0.3.0 — same step 0 through step 6, same panel rubric, same syntax linter, same opt-out semantics. The pivot is additive: the outer doc skill is new, the inner skill is preserved.
- Open questions from v0.3.0 (subagent vs same-context, archetype tie-break, regenerate vs edit-in-place, etc.) remain resolved per their v0.3.0 dispositions; the doc-level skill inherits the same answers where applicable.
- One genuinely new open question: how the doc design pass's "is this complex enough to need a Lifecycle / state-and-persistence / glossary section?" decision performs in real use. Empirical only.

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
