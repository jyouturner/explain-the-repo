# Tutorial — building `explain-the-repo`, commit by commit

A walkthrough of how this skill got built across 17 commits. Each phase: what was done, why, what surfaced, and what got committed. The point is to show *the cadence and the empirical loop* — most "design choices" came out of running small experiments, not deliberation.

This pairs with [`lessons.md`](lessons.md) (the seven insights extracted from this journey) and [`checklist.md`](checklist.md) (the operational version of those insights). Read this if you're learning the methodology by example.

The full git log is at the bottom of this doc for reference.

---

## Phase 1 — Initial skill (v0.1 → v0.2)

**Commits:** `d7d5961`, `e15ec0c`, `27cf6e6`

**Goal.** Produce a focused skill called `system-diagram-in-mermaid` that draws legible Mermaid system diagrams. Address the canonical AI-generated-diagram failure modes (noun inventory, layer cake, decorative color, cross-cutters drawn as peer boxes).

**Shape.** A 5-step procedure in `SKILL.md` (pick the diagram's job → plan the trace → map to Mermaid → render with elk → handle iteration mode). Two reference files (`diagnostic-checklist.md`, `mermaid-patterns.md`). One worked example (`examples/walkthrough.md` for a multi-tenant RAG service).

**What was right.** Selectivity — not every component as a box; semantic axis for color; cross-cutters as edge labels. The procedure prevented the worst failure modes when followed.

**What was missing.** No mechanism to *check* the output. The skill was guidance; if the model drifted, nothing caught it.

---

## Phase 2 — Panel critique design + hand-run validation

**Commit:** `bf1c775` — *add panel-critique refinement loop: design + hand-run prompt*

**Trigger.** A user observation: "beautiful diagram" is subjective; what does "good" actually mean? Karpathy's framing — "channel many perspectives, don't ask 'you'" — suggested simulating a panel of personas rather than a single LLM judge.

**Decision: build the design doc and a hand-run artifact before any automation.** `design/refinement-loop.md` proposed a four-panelist critique (Trace Reader, Visual Encoding Critic, Scope Steward, archetype-specific Domain SME) with disjoint rubrics. `references/panel-prompt.md` was the hand-runnable artifact: paste it into a session with a diagram and get back a JSON report.

**Validation: stage 1 hand-runs.** Six panel critiques across three real repos plus a sanity check on a deliberately bad diagram. Trace Reader was silent on every with-skill output (good — confirmed the procedure prevents basic structural failures). The other panelists found 1–4 real issues per diagram. False-positive rate was ~1 per diagram.

**What was learned.** Disjoint rubrics held; lane discipline worked; the panel surfaced real issues. *But:* dedup was unreliable on borderline cases (two panelists flagging the same span under different anchors), and single-run stochasticity was real (different runs surfaced different subsets of issues against the same diagram).

**Gate to phase 3.** "Does the panel earn its tokens?" → empirically, yes, on with-skill outputs. Move forward with integration.

---

## Phase 3 — Phase C integration (panel as default)

**Commit:** `9eaecd3` — *ship phase C: integrate panel critique into default skill flow*

**Decision.** Bake the panel into `SKILL.md` as a default step 6, with a filter table partitioning issues into revision-worthy (auto-revise once) and borderline (surface in panel summary). Documented in `design/integrated-flow-sketch.md`.

**Filter table calibration.** Picked initial assignments based on stage-1 evidence. `noun-inventory` (any band) classified as revision-worthy. Spawn a subagent for the panel critique (independent context to avoid self-grading bias).

**Cost note.** Token cost moved from 1× → 2–3× the no-panel baseline (generator + panel + optional revision).

---

## Phase 4 — First real-repo worked examples (calibration)

**Commit:** `89335a5` — *add three real-repo worked examples (calibration outputs)*

**What was done.** Took three real repos (autoresearch, recs_two_tower, graphql-node) and produced diagrams under the new skill. Saved each as a worked example.

**Surface during calibration.** The autoresearch reviser regressed: the panel flagged `noun-inventory` at 22 nodes, the reviser collapsed four orchestrator nodes (the `discover` → `triage` → `spec` orchestrated path) into a single `dd` node — which then had `*`-separated subtitles burying the orchestrated-vs-single-call architectural choice. We *fixed one anchor and created another* (`choices-buried`).

**Why this mattered.** The architecture (panel + filter + bounded revision) was right. The *threshold classification* was wrong: hard-fail `noun-inventory` shouldn't be revision-worthy because count-driven consolidation buries plan-named architectural choices.

**Resolution.** Reclassified `noun-inventory` (any band) as borderline. Surface the count to the user; let them decide between consolidation and splitting into sibling diagrams. (See `design/integrated-flow-sketch.md` § Calibration findings for the full record.)

**Lesson.** [`lessons.md` §3](lessons.md) — the architecture is usually right; the bugs are in the filter table.

---

## Phase 5 — Two-runs-unioned for stochasticity

**Commit:** `b374a2a` — *step 6: spawn two panel critiques in parallel, union the issues*

**Trigger.** Stage 1 found that consecutive panel runs against the same diagram returned different subsets of issues. The recs_two_tower `out-of-scope-sprawl` flag (offline pipeline drawn despite being declared out of scope) was caught in round 1, missed in round 2 against the same prompt. Single-run results aren't authoritative.

**Decision.** Step 6 now spawns two panel-critique subagents in parallel and unions their flagged issues (collapsed by `(anchor, quoted span)`) before applying the filter. Wall-time cost is unchanged (parallel); token cost roughly doubles for the panel step.

---

## Phase 6 — Real bug + the syntax linter (Layer B)

**Commits:** `19a968f` (the bug fix), `824a227` (the linter that prevents the next one)

**The bug.** A user observed that `examples/real-repos/graphql-node.md` failed to render on GitHub: "Unable to render rich display." Root cause: `norm -.->|products[]| resolver` had unquoted `[]` in an edge label, which Mermaid v11's parser reads as opening a node-shape declaration.

**Immediate fix.** Quote the label (`|"products[]"|`). Update `references/mermaid-patterns.md` § Syntax footguns to mention bracket characters explicitly (the existing footgun #6 only covered parens and `>`).

**The deeper question.** Why did this ship past a perfectly good "syntax footguns" reference? *Because the reference was passive* — the model could read it but didn't, at the moment of failure. Documentation isn't enforcement.

**Layer B.** Add a syntax-linter subagent (`references/syntax-lint-prompt.md`) that runs alongside the panel critique. The linter is mechanical: scans for parser-blocking footguns (unquoted `[]`/`()`/`{}`/`@`/`>` in edge labels, reserved keywords as IDs, the markdown-list trap, linkStyle out-of-range), returns auto-fixes as `(find, replace)` pairs. Step 6 now spawns *three* parallel subagents: 2 panel critics + 1 linter.

**Validation.** Tested against the actual `products[]` case — linter caught it correctly with the right find/replace pair, no false positives on stylistic-but-parseable patterns elsewhere.

**Lesson.** [`lessons.md` §2](lessons.md) — when tempted to *document* a failure mode, ask whether it should be a *check*.

---

## Phase 7 — Step 0 design pass (multi-diagram sets)

**Commits:** `76db167` (step 0 added), `6d6ec00` (autoresearch regenerated), `00ecd74` (open questions resolved)

**Trigger.** A user observation: the autoresearch diagram is sprawling because the system genuinely has multiple aspects (5-phase pipeline + cross-run feedback loop + persistent state landscape) that don't fit in one frame. The panel correctly flagged the sprawl, but auto-consolidation buried what mattered. The right answer wasn't fewer nodes; it was *multiple diagrams*.

**Decision.** Add a step 0 — *diagram-set design pass*. Decide whether the system warrants 1 diagram (default) or up to 5 siblings (cadence siblings, zoom siblings, topology siblings, lifecycle siblings, failure siblings). Sibling rules in `references/diagram-set-design-pass.md`. A new set-level critique (`references/diagram-set-panel-prompt.md`) reviews the set composition when N > 1 — separate from the per-diagram panel.

**Regenerated autoresearch as a 4-diagram set:** request trace headline + Phase-2 zoom + cross-run feedback cadence sibling + persistent-state topology sibling. Each diagram is clean within its own scope (~6–13 nodes). Together they cover what the original 22-node single diagram couldn't.

**Open-questions cleanup.** `00ecd74` resolved five questions inline in `SKILL.md`: opt-out path (user signals trigger N=1 + skip panel; linter still runs), iteration-mode-redraw (route back to step 0; original is often a single-diagram-overreach symptom), archetype tie-break across two parallel runs, revise contract (regenerate from step 2; never edit Mermaid in place), subagent vs same-context (subagent — settled).

**Cost.** Multi-diagram sets pay 1 + N×4..5 calls. For N=4 worst case: ~21 calls. Real cost commitment, but only paid when the system warrants it; simple systems stay at N=1.

---

## Phase 8 — The pivot to `explain-the-repo`

**Commit:** `7ca6680` — *rename: system-diagram-in-mermaid → explain-the-repo (v0.4.0)*

**Trigger.** A user observation: looking at the multi-diagram autoresearch example, the *deliverable* isn't a diagram — it's an architecture document containing diagrams plus prose framing each one. The diagrams are means; the doc is the end. The skill name and positioning had been wrong since v0.1; the work was right.

**Decision: full rename, not incremental reframe.** The user explicitly chose option C (full pivot) over option A (light reframe) or option B (companion skill alongside the existing one). Reasoning: incremental reframes leave conceptual cruft; a clean rename forces the architecture to actually fit the new positioning.

**What changed.** SKILL.md restructured around the new outer loop (Phase A doc design pass → Phase B per-section generation → Phase C doc-level review). New reference files: `references/diagram-procedure.md` (the existing 6-step procedure lifted intact as the inner loop), `references/doc-design-pass.md` (section taxonomy for Phase A), `references/doc-panel-prompt.md` (doc-level coherence critique). Renamed: `design-pass.md` → `diagram-set-design-pass.md` for clarity.

**What stayed the same.** The inner diagram procedure — every single thing we built in phases 1–7 — was preserved unchanged as the inner loop. The diagram-set design pass became the inner step 0; the per-diagram panel and syntax linter became the inner step 6. None of the work was wasted.

**Validation.** `51bdf4e` — first cold-run end-to-end on `~/Github/enterprise` (an AI-agent framework with HR and retail reference implementations the skill had never seen). Real subagent loaded SKILL.md + all references, read the repo, produced a 7-section architecture doc with 4 diagrams. The syntax linter caught 3 actual rendering bugs (cylinder labels with unquoted `<>` and `{}` placeholders); independent linter re-run on the final file confirmed all 4 diagrams clean post-fix.

**Lesson.** [`lessons.md` §5](lessons.md) — pivot on artifacts, not suspicion. The autoresearch case made the right positioning visible.

---

## Phase 9 — Cold-run regenerations + README rewrite

**Commits:** `e95b40c` (URL cleanup post-rename), `408bbbf` (3 cold runs), `44145a0` (README rewrite)

**Cold-run regenerations.** The autoresearch / recs-two-tower / graphql-node examples in `examples/real-repos/` had been hand-shaped during the v0.4 pivot using prior knowledge of those repos. They documented what the skill *should* produce, but they hadn't been produced *by* the skill. `408bbbf` replaced them with cold-run outputs from three parallel subagents that loaded the skill files, read each repo from scratch, and produced full architecture docs end to end — same pattern that produced `enterprise.md`.

**Real findings from the cold runs:**
- recs-two-tower surfaced a real codebase inconsistency as a TODO: `src/inference.py:72` reads `preprocessor_data.pkl` but `src/training.py:212` only writes `preprocessor.json`. The "don't fabricate, mark TODOs" rule paid off concretely.
- graphql-node surfaced two README-vs-code mismatches: the `rate(1 minute)` scheduled-event intent was unclear, and the `mssUtils.PROMISE_UTILS.execAll` example in the README didn't match current resolver code. Both marked as TODOs rather than papered over.
- autoresearch's syntax linter caught one auto-fix (a decision label with `*`/`/`/`<br/>` that needed quoting) and surfaced one prose typo. Cited line numbers were verified by `grep`/`Read` during exploration.

**Independent linter validation.** Ran the syntax linter as a separate subagent against all 14 diagrams across the four cold-run files. All clear. Confirmed the linter catches real bugs *during generation*, and the final files are parser-clean.

**README rewrite.** A user observation: the README led with "what this fixes" (a meta narrative about failure modes), not with what users actually need to know. `44145a0` rewrote it around three goals: (1) what it is (one sentence), (2) what you get (the four cold-run examples linked at the top), (3) how to use it (install + canonical prompt). Cut from 267 lines to 67. The old content (Before/After example, full Rendering caveats table, etc.) was redundant with the worked examples.

---

## What you can take to the next skill

Read [`lessons.md`](lessons.md) for the seven insights and [`checklist.md`](checklist.md) for the operational version. The four most-leveraged in retrospect:

1. **Calibrate on real artifacts before shipping any threshold-driven behavior** (Phase 4's reviser regression).
2. **Convert "documented failure modes" to "procedural checks" wherever mechanical** (Phase 6's syntax linter).
3. **Design the opt-out alongside any second or third critique pass** (Phase 7's late opt-out addition).
4. **Pivot positioning when the artifact tells you, not when you suspect** (Phase 8's rename).

And the meta: most "design choices" in this skill came from running small experiments, not deliberating. When you find yourself debating whether something will work, ask whether you can spend 15 minutes finding out instead. Usually you can.

---

## Reference: the full git log

```
44145a0 rewrite README around three user goals: what it is / what you get / how to use
408bbbf regenerate three real-repo examples via cold runs of explain-the-repo
e95b40c point install paths and issue link at the renamed GitHub repo
51bdf4e add enterprise example: cold end-to-end validation of explain-the-repo
7ca6680 rename: system-diagram-in-mermaid -> explain-the-repo (v0.4.0)
00ecd74 resolve open questions and align all surfaces with phase C + step 0
6d6ec00 regenerate autoresearch example as multi-diagram set
76db167 add step 0 design phase for multi-diagram sets
824a227 add syntax-lint subagent to step 6 (Layer B)
19a968f fix unquoted [] in edge label; document the bracket footgun
b374a2a step 6: spawn two panel critiques in parallel, union the issues
89335a5 add three real-repo worked examples (calibration outputs)
9eaecd3 ship phase C: integrate panel critique into default skill flow
bf1c775 add panel-critique refinement loop: design + hand-run prompt
27cf6e6 add examples and eval
e15ec0c Generic model IDs in cost-routing example
d7d5961 Initial commit: system_diagram_in_mermaid skill
```
