# Doc design pass — picking the architecture-doc sections

The skill produces an architecture-doc *section set*, not a free-form essay. This file is the procedure for deciding what sections the doc needs before any prose or diagrams are generated. Default doc shape is three sections (headline + architecture overview + where-to-start); you add sections only when the system has properties the default shape can't honestly cover.

This is Phase A / step 0 of `SKILL.md` — runs before per-section generation.

---

## What an architecture doc is

A single markdown file (~300–800 lines) that explains a codebase to a stranger. It's not a README (which is for end users), not a code-level reference (which is for callers of specific functions), not an ADR (which is for a single decision). It sits between high-level "what is this product" and low-level "what does function `foo()` do" — at the **system explanation** level.

Its goal: someone who's never seen this repo can read the doc once, then navigate the codebase with a working mental model of what's where and why.

Crucially: the doc is selective. A doc with 12 sections becomes a documentation site, not a single doc. Cap at 8–10 sections.

## The section taxonomy

These are the section types the skill knows how to produce. Pick from this list.

### Always include

1. **Headline (one paragraph).** What this system is, in 2–3 sentences. Names the system, its purpose, and the most architecturally interesting property. *Not* a feature list. Always at the top.

2. **Where to start reading (file pointers).** 3–6 bullets, each with a path and one sentence. Where are the entry points, the key interfaces, the file that explains the most. This is the single highest-leverage section for a stranger. Always include.

3. **Architecture overview (diagram-set or hybrid).** The headline explanation of how the system works at the system level. Almost always involves a diagram (or a multi-diagram set if the system has cadence siblings, zoom siblings, etc. — see `references/diagram-set-design-pass.md`). May include prose framing the diagram(s).

### Add when warranted

4. **Component summaries (prose, optionally with one diagram per component).** Per-major-component description: responsibility, surface (what it exposes), boundary (what it doesn't do), key files. Add when the system has 3–7 distinguishable components and the architecture overview alone doesn't carry enough detail. Skip for single-component systems. **Use the required template below — see "Component summaries — required template."**

5. **Data flow (prose + diagram-set or hybrid).** How data moves through the system over time — different from the request flow shown in the architecture overview. Add when data is the load-bearing concept (ML pipelines, ETL, event-driven systems, agents that produce and consume their own state). Skip for stateless request handlers where data flow == request flow.

6. **State and persistence (prose + diagram, often topology-shaped).** What persists between requests / runs / sessions, where it lives, what reads / writes it. Add when there's nontrivial persistent state spread across multiple stores. Skip for stateless services or single-database systems where the architecture overview already shows it.

7. **Lifecycle (prose, sometimes diagram).** Build / deploy / runtime distinctions, especially for ML systems with offline training + online serving, or systems with build-time and run-time concerns that are structurally different. Add when the system has a clear lifecycle split.

8. **Failure modes / runbook (prose).** Known gotchas, common failure patterns, what to look at when things break. Add when the user explicitly wants operations focus, OR when the codebase has well-documented failure handling worth surfacing. Skip for greenfield repos.

9. **Test and eval surface (prose).** Where the system's correctness gets verified. Names where the eval lives, how to run it, what the metrics mean, what passes vs fails. Add when the repo has a non-trivial verification mechanism beyond standard unit tests — a golden set, a benchmark harness, a regression suite, an observability layer for model quality. Skip when test setup is trivial (a `pytest` invocation that needs no explanation) — but do not skip just because the eval is small; a 12-question hand-curated golden set is exactly the kind of thing a stranger needs to know exists before changing anything.

10. **Glossary (prose).** Domain terms, internal jargon, project-specific abbreviations defined once. Add when the codebase has heavy domain language (financial, medical, scientific) or when the system has ≥ 5 named identifiers (component IDs, type discriminators, state field names) a stranger won't know. Skip for general-purpose CRUD.

11. **Out of scope (prose, always last).** What this doc deliberately doesn't cover, and where to find that information instead. One or two sentences each. Always include — even a one-line "The deploy pipeline is documented in `infra/README.md`" is enough.

### What NOT to include in this skill's docs

- **Install instructions** — those belong in README.md.
- **API reference** — generated by OpenAPI / typedoc / similar.
- **Tutorials and how-tos** — separate doc shape, separate audience.
- **Code style / contribution guidelines** — `CONTRIBUTING.md`.
- **License / copyright** — repo metadata.

## Decision rules

Default doc shape: **headline + architecture overview + where-to-start** (3 sections).

Add sections per these rules. Each rule is one sentence + the trigger.

1. **Component summaries** if the system has 3–7 distinguishable major components AND the architecture overview alone wouldn't tell you what each one is responsible for.
2. **Data flow** if data is the load-bearing concept (the system's job is *what happens to data* more than *what the user sees*).
3. **State and persistence** if there's nontrivial persistent state (multiple stores, hash-chained logs, cross-run state) that the architecture overview can't fully represent.
4. **Lifecycle** if the system has a clean build-time vs run-time split (ML training + serving; CI build pipeline + runtime service; etc.).
5. **Failure modes / runbook** only if the user asks, OR if the codebase already has structured failure-mode documentation worth surfacing.
6. **Test and eval surface** if the repo has an eval set, golden file, benchmark harness, or any non-trivial verification mechanism beyond standard unit tests. The signal: there's a `data/eval/`, `evals/`, `benchmarks/`, or similar directory with hand-curated content; or there's an `eval.py` / `benchmark.py` / `run_eval.sh` entry point; or the README headlines a metric (hit@k, accuracy, NDCG, F1, etc.). If any of those, include the section.
7. **Glossary** if the codebase has ≥ 5 domain-specific terms a stranger wouldn't know.

### Merge rule: lifecycle and state-and-persistence

These two sections overlap heavily for ML, data-pipeline, and any system where the offline lifecycle's outputs *are* the persisted state the online path reads. Decide before generation:

- **Merge into a single "Lifecycle and persistence" section** when the build-time output of one stage is the run-time input of another, AND the persistence topology you'd draw is essentially the lifecycle diagram you'd draw with a different framing, AND removing either one would leave the other unable to make its point.
- **Keep separate** when persistence is incidental to lifecycle (e.g., a stateless service that writes to a transient cache), or when lifecycle is build-vs-deploy (not data-flow-shaped) and persistence is about runtime stores unrelated to the build.

The criterion is the "remove one, does the other still make sense" test. If the answer is no — merge.

Skip sections that don't earn their place. A 4-section doc that's all load-bearing is better than a 9-section doc with three padding sections.

## Cap rules

- 8–10 sections is the practical ceiling for a single doc.
- Beyond 10 sections, ask the user: "do you want a doc set (separate files for distinct audiences) or a leaner single doc?" — don't unilaterally produce a 14-section doc.
- A section with fewer than 3 paragraphs of unique content should usually be folded into another section.

## Component summaries — required template

When the doc plan includes a Component summaries section, every component subsection uses this shape unless the components are heterogeneous enough that a uniform shape would mislead (rare — call it out in the section header when skipping):

```
### `<component_name>` — <one-phrase role>

<Responsibility paragraph: 2–4 sentences. What does this do, and why does
it exist as its own component instead of being part of something else?>

**Surface**: <what this component exposes — CLI invocation, function
signature, API endpoint, message contract. Specific.>
**Boundary**: <what this component deliberately doesn't do. Names the
line. Specific.>
**Key files**: <2–4 file paths, with line ranges where useful>.
```

The **Boundary** line is the highest-leverage part. It forces the agent to articulate negative space — what the component *isn't* responsible for — which prevents the next reader from looking for capabilities in the wrong place. Skip it only if the component genuinely has no meaningful boundary to name (rare — most do).

The **Surface** line should be the thing a caller of this component would invoke or import. Not a description of "what it does" (that's the responsibility paragraph) — the literal entry point.

The **Key files** line points the reader at where to read next. If a component spans many files, pick the 2–4 that explain the most. Cite line ranges (`src/foo.py:30–80`) when a component's most interesting logic lives in a specific block.

## Visual vocabulary lock-in

After the section list is decided and before any diagram is generated, lock in a visual vocabulary that all diagrams in the doc will share. This prevents the reader from re-learning a color/shape mapping in every section.

Pick three things:

1. **Color palette** — 4–6 colors, each with an explicit semantic meaning that holds across every diagram in the doc. Example for a data-pipeline system:

   - yellow `#fef3c7` = persisted data (anywhere it appears)
   - blue `#dbeafe` = code / components
   - purple `#ede9fe` = user-facing I/O / system boundaries
   - green `#d1fae5` = online / serve cadence (only when cadence is a recurring axis)
   - red `#fee2e2` = failure paths or sparse-modality (only when relevant)

   Use the same palette across every diagram. Do not pick a fresh palette per diagram.

2. **Shape vocabulary** — 2–4 shapes, each with an explicit meaning. Default for system-explanation docs:

   - `[Box]` = code component
   - `[(Cylinder)]` = persisted storage
   - `(["Stadium"])` = user-facing I/O / start/end nodes
   - `{{Hexagon}}` = transformation / algorithm (use sparingly; renders poorly in dagre — see `references/mermaid-patterns.md` § "Publishing to GitHub README")

   Avoid mixing shape conventions within the doc. If cylinders mean storage in diagram 1, they mean storage in every other diagram too.

3. **Stable axis assignments** — the load-bearing claim. State explicitly: "yellow always means persisted data in this doc," "stadium shapes always mean external I/O." This is the constraint each per-diagram generation must hold to.

Pass the locked vocabulary as input to each per-section diagram-procedure invocation. The diagram procedure's step 2 (plain-text plan) must declare which colors/shapes from the vocabulary it will use, and confirm that none of its choices conflict with prior diagrams in the same doc.

The doc-panel critique enforces this via the `inconsistent-vocabulary` anchor — see `references/doc-panel-prompt.md`.

## Format for the doc plan

Produce this as a short prose block before generating any prose or diagrams:

```
DOC PLAN — <System name>: <one-line system description>

Existing docs detected (and what they cover):
- <path>: <one-line summary of what this doc already covers>
- ...

Sections (in order):
1. [headline] Headline paragraph: <what the headline names>
2. [where-to-start] Where to start reading: <which entry points and key interfaces>
3. [architecture-overview, diagram-set] Architecture overview: <which diagrams; will invoke
   the diagram procedure with N=<expected diagram count>>
4. [component-summaries, prose] Component summaries: <which components, why each>
5. [data-flow, hybrid] Data flow: <prose + diagram(s) or just prose>
...
N. [out-of-scope] Out of scope: <what's NOT covered, where to find it>

Sections deliberately omitted (and why):
- <section type>: <reason — including overlaps with existing docs>

Visual vocabulary (locked for the whole doc):
- Colors:
  - <color hex> = <semantic meaning that holds across every diagram>
  - ...
- Shapes:
  - <shape> = <semantic meaning>
  - ...
- Stable axis assignments: <one sentence per axis>

Estimated section count: <N>. Reasoning: <one sentence — which decision rules applied>.
```

The "Existing docs detected" item is populated by Phase 0's existing-docs scan (see `SKILL.md` Phase 0). When the proposed doc would substantially overlap one of these, the "Sections deliberately omitted (and why)" item should explicitly name the overlap.

The "Sections deliberately omitted" item is load-bearing. If you skipped Failure Modes, Test-and-eval-surface, or Glossary, name the choice so the doc-panel can sanity-check it.

The "Visual vocabulary" block locks the color/shape mapping for the entire doc — see "Visual vocabulary lock-in" above. Without this block, downstream diagrams will pick their own colors and the reader has to re-learn each diagram's mapping.

## Surface to the user before generating

Show the doc plan to the user and ask for a quick sanity check before generation begins.

> "I'll produce a 5-section architecture doc: headline + where-to-start + architecture overview (3-diagram set covering request flow, state map, and cross-run feedback) + component summaries + out of scope. Skipping data-flow (covered in the architecture overview's diagrams) and failure-modes (no operations focus in your request). Sound right before I generate?"

If the user dismisses the design pass ("just give me a quick overview"), respect that — drop to the minimum 3-section shape and skip Phase C's doc-panel.

## When the doc design pass is over

You have:
- A list of sections (3–10), each with a type (`prose`, `diagram-set`, `hybrid`) and a one-line scope.
- Source files each section grounds in.
- A list of section types deliberately omitted, with reasons.

For each section in the plan:
- **Prose sections** are written in Phase B. Cite paths. Don't fabricate.
- **Diagram-set sections** invoke `references/diagram-procedure.md` for their visual content. The diagram procedure handles its own design pass, generation, and critique within the section's scope.
- **Hybrid sections** combine both — write prose first, then generate diagrams in the same section.

The doc-panel critique (`references/doc-panel-prompt.md`) runs once on the assembled doc in Phase C — only when the doc has more than 3 sections. Three-section docs and opt-out docs skip the doc-panel.
