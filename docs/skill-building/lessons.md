# Lessons learned — building `explain-the-repo`

Seven insights from the journey of building this skill, plus a meta-observation about cadence. Not generic skill-building advice — each lesson here is tied to a specific incident or failure mode we hit. The companion documents are [`checklist.md`](checklist.md) (operational version of these lessons) and [`tutorial.md`](tutorial.md) (the journey itself, commit by commit).

The journey: started as a focused skill for legible Mermaid diagrams, grew through an empirical panel-critique design, hit a real rendering bug that drove a syntax linter, calibrated against real diagrams that exposed filter-table bugs, fanned out into multi-diagram sets for complex systems, and finally pivoted from "draw a diagram" to "produce an architecture document." 17 commits, three real-repo cold-run validations, one rename.

---

## 1. Disjoint rubrics make multi-persona critique work

Karpathy's framing — "channel many perspectives" rather than asking the LLM "what do you think?" — is real but only works in practice when each persona has a non-overlapping scope. Trace Reader, Visual Encoding Critic, Scope Steward, and an archetype-specific Domain SME each got their own anchor list and an explicit "out of scope for this panelist" line. Without those fences, four "panelists" collapse to one generic critique that averages everyone to the same vague voice.

The mechanism that actually enforced disjointness was anchoring each panelist to specific failure-mode IDs from a shared checklist (e.g. `missing-return-arrow`, `wrong-trust-surface`, `out-of-scope-sprawl`). The Trace Reader's rubric *can't* fire on color-encoding issues because color isn't in their anchor list. That structural separation is what makes the critique multi-perspectival rather than generic.

Implication for other skills: if you're using multiple personas / agents / critics, the design work is in the rubric scoping, not the personality writing.

## 2. Passive references get ignored at the moment of failure

The Mermaid-patterns reference had a "Syntax footguns" section that warned about unquoted brackets in edge labels. The skill referenced it. The model could read it. And `products[]` shipped anyway in `examples/real-repos/graphql-node.md`, breaking GitHub rendering with "Unable to render rich display."

The fix wasn't more documentation. It was a procedural linter — a subagent that scans the source and returns `(find, replace)` pairs to apply verbatim. The linter doesn't "remember" footguns; it scans for them every time. Reading is decoration; scanning is enforcement.

Implication: whenever you're tempted to *document* a failure mode the model should avoid, ask whether it should be a *check* instead. The cost of writing a small linter is comparable to writing a "common pitfalls" doc, and the linter actually catches bugs.

## 3. Calibrate on real artifacts; the bugs are in the filter table

The autoresearch reviser regression — collapsing four orchestrator nodes into one to satisfy a `noun-inventory` count, which then created a fresh `choices-buried` violation — wasn't predicted by the original filter table. We had reasoned about it. We had written design docs. The regression only surfaced when we actually ran the loop end-to-end on a real diagram.

The architecture (panel critique, revision-worthy vs borderline split, bounded one-round revision) was right. The *thresholds and classifications* (specifically, "is hard-fail noun-inventory revision-worthy or borderline?") were wrong. Five hand-runs revealed that — the same five hand-runs would have been hard to motivate on theoretical grounds before they revealed something.

Implication: design the architecture upfront, but don't ship without calibrating thresholds against real artifacts. Almost every meaningful design fix in this skill came from running the loop end-to-end, not from another design-doc iteration.

## 4. Token cost compounds invisibly; design the opt-out before the third critique

We went from 1 model call per request (v0.1 — generator only) to a worst case of ~21 calls in a 4-diagram set under v0.4 (generator × 4, panel critic × 8, syntax linter × 4, doc-panel × 1, doc-revision × 1, plus per-diagram revisions). Each step felt incremental: add a panel critique here, run it twice for stochasticity, add a syntax linter there. The total wasn't.

The opt-out path (`"quick overview, no critique"` / `"no panel"` / `"rough sketch"`) got added late, after the cost was already real. Adding it earlier — alongside the *first* layered critique pass — would have been cheaper to design and would have saved the awkward retroactive integration.

Implication: if you find yourself adding a second or third critic, design the bypass at the same time. The cost of a "skip the panel" path is small upfront and large to retrofit.

## 5. The work outgrows the positioning. Pivot on artifacts, not suspicion

The skill started as `system-diagram-in-mermaid` — produce a single Mermaid system diagram. Every iteration revealed that what users actually wanted was an *architecture document containing diagrams*, not a diagram alone. We could have suspected this earlier; we didn't pivot until the autoresearch case made it obviously concrete (a 21-node single diagram trying to cover what should honestly be a multi-section doc with sibling diagrams).

The pivot to `explain-the-repo` was straightforward because the diagram work *became the inner loop*. None of it was wasted. Premature renaming (renaming because we suspected the positioning was wrong) would have created churn; late renaming (renaming once the artifact made it obvious) was clean.

Implication: when iterating on a skill, watch the artifacts it produces. The right positioning is usually visible in what users find useful, not in what the skill claims to do. Pivot when the artifacts show you, not when you suspect.

## 6. Tell the model what to do when it doesn't know

The instruction "don't fabricate; mark `TODO: confirm in <file>`" caught three real codebase inconsistencies across the cold-run validations: the `preprocessor.json` vs `preprocessor_data.pkl` mismatch in `recs_two_tower`, the README-vs-code drift in `graphql-node` (the `mssUtils.PROMISE_UTILS.execAll` example didn't match current resolver code), and the unclear intent of `graphql-node`'s `rate(1 minute)` scheduled event.

Without that explicit fallback, the model would have produced confident fiction — plausible-sounding answers that match the surrounding pattern but aren't grounded in the actual code. Each of those TODOs is more useful than a confident wrong answer would have been; the user can decide whether to investigate.

Implication: any skill that asks the model to assert facts about a real artifact (a codebase, a database schema, a system) needs an "I don't know" exit ramp built in. The shape: "if you can't verify X from your reading, mark `TODO: <specific phrasing>` and cite the file you couldn't verify it from." That single instruction prevents a whole class of fabrication failures.

## 7. Live design docs are load-bearing

`design/refinement-loop.md` and `design/integrated-flow-sketch.md` got updated continuously through the project — not just at the start, but as calibration findings rolled in, as open questions got resolved, as post-ship adjustments landed. They have date-stamped sections and explicit "resolution notes" that connect each design choice to the evidence that produced it.

Six months from now, when someone asks "why is `noun-inventory` hard-fail in the panel-prompt's rubric but borderline in the integrated flow's filter table?", the answer is in the calibration findings section of `integrated-flow-sketch.md`. With the evidence. Without that, the design choice looks arbitrary and gets re-litigated.

Implication: design docs are most valuable when they capture the *journey* — what was decided, what evidence, what was tried and abandoned. Static design docs that get written once and never updated rot in weeks. The maintenance cost of keeping a design doc current is small relative to the cost of someone (you, in three months) re-deriving the rationale.

---

## The meta-observation: cadence over deliberation

The pattern that produced this skill — *small concrete experiments, validated cheaply before committing to bigger changes* — is itself the methodology worth keeping.

Almost every "should we do X?" question got answered by spending 10–20 minutes running a real example, not by another round of design-doc debate. The cost of empirically testing a small choice was usually lower than the cost of arguing about it.

A few examples:
- "Does the panel work?" → run it once on a deliberately bad diagram (~10 min).
- "Will the corrected filter table fix the autoresearch regression?" → re-run the same reviser without `noun-inventory` (~3 min).
- "Does the doc skill work cold?" → spawn one subagent against the enterprise repo (~10 min, ~150K tokens).
- "Are the three hand-shaped examples fair representations?" → three parallel cold runs to find out.

Each of those checkpoints could have been a 500-line design doc justifying the choice in advance. None of them was. The empirical answer was both faster and more honest.

There's a counter-failure mode: running tiny experiments without a coherent picture of where you're going. The two protect each other. The design docs (live, updated, evidence-attached) keep the experiments aligned. The experiments keep the design docs honest. Either alone produces drift; together they produce a skill that actually works.

If there's one thing to take from this journey to the next skill: when you find yourself debating whether something will work, ask whether you can spend 15 minutes finding out instead. Usually you can. And usually the answer is more interesting than the debate would have been.
