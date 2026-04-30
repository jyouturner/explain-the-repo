# Diagram procedure (inner loop)

The procedure for generating a Mermaid diagram set, used by Phase B of the `explain_the_repo` skill for diagram-set sections, and usable standalone for "draw a diagram of X" requests.

This is a self-contained procedure: design pass for the diagram set, per-diagram planning and rendering, panel critique, syntax linter, and bounded revision. It produces *one diagram set* (1–5 sibling diagrams) with their plans, sources, notes, and panel summaries.

## When to use

- Inside Phase B of `SKILL.md`: a section the doc plan classified as `diagram-set` or `hybrid` invokes this procedure to produce its visual content.
- Standalone: "draw a diagram of X", "request flow for Y", "show me how Z works". The user wants visual representation alone, not a full architecture doc.

## What this procedure produces

For a single invocation:

- A **diagram set** of 1–5 diagrams (default 1; complex systems warrant siblings).
- For each diagram in the set: a plan, the Mermaid source, notes, and a panel summary.
- A diagram-set design summary (when N > 1) and a diagram-set panel summary (when N > 1).

The output is ready to embed in a markdown section or return standalone.

## The procedure

Six steps. Step 0 is the diagram-set design pass; steps 1–5 run *per diagram* in the set; step 6 is the per-diagram critique. Don't skip the planning steps and jump to Mermaid.

### 0. Diagram-set design pass

Before drawing anything, decide the set composition. Read `references/diagram-set-design-pass.md` in full — it contains the sibling rules (when to add a cadence sibling, zoom sibling, topology sibling, lifecycle sibling, failure sibling), the cap rules (≤ 5 diagrams; sub-3-node siblings go in NOTES instead), and the format for the set design.

Default set size is 1. Add siblings only when the system has properties that genuinely don't fit one frame.

Produce the set design as a short prose block. Show it to the user (or to the doc-skill caller) and ask for a quick sanity check before any drawing begins. If the user dismisses the set design ("just give me one diagram"), respect that — drop to N=1 and use the headline only.

If N > 1, run the set-level critique once: spawn a subagent that reads `references/diagram-set-panel-prompt.md` and returns a JSON report on the set composition. Apply revision-worthy issues (`missing-aspect`, `wrong-cadence-split`, `unanchored-zoom`) by adjusting the set design before generation; surface the rest in the set-level summary.

If N = 1, skip the set-level critique — the per-diagram panel critique (step 6) covers single-diagram cases.

After step 0 you have:
- A set of N diagrams (1 ≤ N ≤ 5), each with archetype, scope, and concrete entry point.
- Named relationships between diagrams in the set.
- A list of aspects deliberately excluded.

For each diagram in the set, run steps 1–6 below. Step 1's archetype is fixed by step 0; step 1 confirms the specific job within that archetype.

### 1. Pick the diagram's specific job

Within the archetype assigned by step 0, decide which of these the diagram is for:

- **Trace** — show one request flowing end to end through the system. Default for most cases.
- **Topology** — show what components exist and what's connected to what, without a specific request flow. Used when the user explicitly wants a map, not a story.
- **Failure / recovery path** — show what happens when something goes wrong. Always a separate diagram from the happy-path trace.
- **Cross-cadence loop** — batch processes, retraining loops, audit aggregation. Different time-scale than the request flow.

Step 0 has assigned the archetype; step 1 confirms the specific instance. *Which* request, *which* failure path, *which* cadence — the concrete entry point that grounds the rest of the planning.

State the specific job in one line at the top of the diagram's section so the user can correct course before you commit to pixels.

### 2. Plan the trace in plain text first

Before writing Mermaid, write a short plain-text outline. The outline has four parts:

1. **Concrete entry point.** Pick a specific, named request — not "a user request" but something concrete like "user submits: 'summarize this week's incidents'" or "checkout: cart of 3 items, signed-in customer, default address." Specificity forces real decisions about what gets drawn.
2. **The path.** List the components the trace touches, in order, as a numbered list. If a component isn't on the path, it doesn't go on the diagram.
3. **The semantic axis.** State what color will encode. Pick exactly one: trust boundary, control plane vs data plane, sync vs async, internal vs external, or category (e.g. "infra / app / data"). Two axes confuse the reader; zero axes makes color decorative.
4. **Out of scope.** Name the things you're deliberately not drawing: failure paths, batch jobs, observability, etc. This is the fence that keeps the diagram from sprawling.

Show this outline to the user and ask for a quick sanity check before drawing.

### 3. Map the architecture into Mermaid primitives

Once the outline is approved, translate it. The mapping is mostly mechanical:

- **Components on the path** → nodes (`[Box]`, `{Decision diamond}`, `[(Storage cylinder)]`, `([Pill])`).
- **Semantic axis groups** → `subgraph` containers, one per group, with a tinted fill via `style`.
- **Component categories within the axis** → `classDef` styles, applied via `class` lines.
- **The trace itself** → solid arrows (`-->`) labeled with what flows on them.
- **Lookups, audit writes, async signals** → dotted arrows (`-.->`) — they're not the main flow, they're side effects.
- **Cross-cutting concerns (PII gates, auth checks, rate limiters)** → labels on the relevant edges, NOT separate boxes. They intercept the trace; they don't participate in it.
- **Architectural choices worth surfacing** (cost-aware tier selection, lazy loading, rules-then-LLM verification) → make them visible structurally — split a box into two, draw both branches with labels — rather than burying them in a subtitle.
- **Sinks** (audit log, metrics) → one node, with multiple incoming dotted arrows. Don't draw the same write four times in four places; consolidate.

For specific Mermaid idioms — subgraph syntax, classDef patterns, linkStyle indexing, elk renderer config — see `references/mermaid-patterns.md`.

### 4. Render with elk, not dagre

The default Mermaid renderer (dagre) produces acceptable layouts for trivial diagrams and tangled layouts for real ones. The elk renderer minimizes edge crossings and handles subgraphs much better. Always configure elk:

```javascript
mermaid.initialize({
  flowchart: { defaultRenderer: 'elk', curve: 'basis', nodeSpacing: 50, rankSpacing: 60 },
});
```

If the user is rendering in an environment that defaults to dagre (GitHub, plain Markdown previews), warn them and either suggest Mermaid Live Editor with elk enabled, or accept the dagre rendering with a note that routing will be slightly worse.

The direction matters. `flowchart LR` (left-to-right) reads naturally for request traces. `flowchart TB` (top-to-bottom) is for hierarchies and stacks. Don't mix — pick one and stick to it.

### 5. If iterating on an existing messy diagram

When the user shows you a v1 / v2 diagram and says "make this better," don't just redraw — diagnose first. Read `references/diagnostic-checklist.md`, identify which specific failure modes the existing diagram exhibits, and tell the user which ones you're going to fix. Then go back to step 0 — the redraw decision is itself a design-phase decision. If the original diagram is sprawling because it's trying to cover what should honestly be N siblings, propose a multi-diagram redraw rather than a "make this single diagram better" reply.

The diagnostic mode is also useful when reviewing someone else's diagram without redrawing.

### 6. Panel critique + syntax lint + bounded revision

Before returning the diagram (or set of diagrams) to the caller, run a one-round panel critique on each diagram. This catches the subtler failure modes the procedure can let through — out-of-scope sprawl, trust-axis violations, buried architectural choices, plan-violation drift.

For each diagram, spawn **three** subagents in parallel:

- **Two panel-critique subagents.** Each reads `references/panel-prompt.md` and applies the four-panelist critique procedure to the diagram independently, in its own context.
- **One syntax-lint subagent.** Reads `references/syntax-lint-prompt.md` and scans the Mermaid source for parser-blocking footguns (unquoted brackets / parens / braces / `@` / `>` in edge labels, reserved keywords as IDs, the markdown-list trap, linkStyle out-of-range). Returns `auto_fixes` as `(find, replace)` pairs plus any `manual_fixes` that need rename-style restructuring.

Independent context matters for the panel runs — do not run the panel in your own context, since you'll grade work you just authored.

When all three subagents return:

1. **Apply linter auto-fixes first.** For each `auto_fixes` entry, replace the `find` text with `replace` verbatim in the Mermaid source. Surface any `manual_fixes` items in the panel summary (these need user attention — typically renames the linter can't safely apply alone).
2. **Union the panel issues across the two panel runs**: collapse pairs that share both anchor and quoted span (one survives), keep all distinct issues. The unioned set is what feeds the partitioning step below. Two parallel runs catch stochastic flag drops.
3. **Resolve archetype tie-breaks.** If the user provided `unknown` as the archetype hint and the two panel runs returned different classifications, prefer the run whose Domain SME persona surfaced concrete domain-specific issues; if both SMEs were silent or flagged similar issues, prefer the first run's classification.

**Opt-out.** If the caller opted out of the panel (per the user's instructions), run only the syntax linter — skip the two panel critique subagents and the bounded revision. Surface "skipped panel critique per user request; syntax linter ran" in place of the panel summary.

Partition the panel's flagged issues into two buckets:

**Revision-worthy** (revise once before returning):

- Trace closure: `no-trace`, `missing-return-arrow`, `dead-end-nodes`
- Cadence and scope: `multiple-cadences`, `out-of-scope-sprawl`, `cadence-unnamed`
- Trust-axis violations: `wrong-trust-surface`, `tenant-shared-confused`
- Iteration-mode failures: `no-diagnosis`, `regression`
- Skill should have stepped aside: `wrong-sublanguage`, `forced-trace`
- Cross-cutter as flow participant: `cross-cutting-as-peer`
- ML serving leak in batch diagram: `serving-leak`

**Borderline** (surface in the panel summary; do not revise): everything else, including `noun-inventory` (any band), `choices-buried`, `decorative-color`, `unlabeled-cross-boundary`, `inconsistent-edge-style`, `gates-invisible`, `wrong-stage-order`. The user is in a better position than a reviser to judge whether these warrant changes.

If revision-worthy issues exist, revise the diagram once. The revision is **bounded to one round** — do not loop. **Regenerate the diagram from step 2's planning step** with the panel issues injected as additional constraints in the plan's "Out of scope" list and step 3's mapping notes. Do not attempt to edit the prior Mermaid source in place — editing risks layering revisions on top of structural mistakes the panel just flagged.

When revising, do not consolidate or remove any node that the plan named as a load-bearing architectural choice, even to address a related anchor. The plan's stated intent takes precedence over count or shape pressure.

If every panelist returned `verdict: ship`, skip the revision and surface a one-line "panel clean" note in the panel summary.

## Output format (when used standalone)

The shape depends on the set size from step 0.

### Single-diagram set (N = 1)

Return four things:

1. **The plan** (from step 2) as a short prose block.
2. **The Mermaid source** in a fenced code block — post-revision if step 6 revised it.
3. **A brief note** about (a) what's out of scope and where it would go in a sibling diagram, (b) any architectural choices that are surfaced visually, and (c) the renderer config.
4. **A panel summary**, at most three sentences. Ship: name the four panelists. Revised: name the issues addressed and any borderline issues surfaced.

### Multi-diagram set (N > 1)

Return:

1. **A diagram-set design summary** (from step 0).
2. **A set-panel summary** (from step 0's critique).
3. **For each diagram in the set**, in the order they appear in the design summary, a section with the four pieces above (plan, Mermaid source, notes, panel summary).
4. **A closing note** if any aspects from the set's "out of scope" list are worth flagging to the user as candidates for follow-up diagrams.

## Output format (when called from the doc skill)

The diagram procedure returns the same content, but the doc skill embeds it inside a doc section's markdown — the section header comes from the doc plan, not from this procedure. Plans, panel summaries, and notes still surface; the doc-skill assembles them into the final markdown.

## Common failure modes to avoid

Even with this procedure, watch for these tendencies:

- **Drawing the noun inventory.** If your diagram has more than ~12 nodes or every component from the source material appears as a box, you've slipped into inventory mode. Cut to the trace, or fan out into siblings via step 0.
- **Layer cake.** Three or more horizontal bands stacked vertically with arrows mostly going down is the layer-cake antipattern. Use subgraphs and explicit return arrows instead.
- **Decorative color.** If your color choices don't map to the semantic axis you named in step 2, they're decorative. Either fix the mapping or go monochrome.
- **Cross-cutting concerns as boxes.** Auth, PII redaction, rate limiting, audit logging, observability — these intercept the flow, they don't participate in it. They go on edges as labels or as gates, not as peer nodes.
- **Architectural choices buried in subtitles.** "LLM Router (cost-aware tiers)" hides the interesting bit. If tier selection matters, draw the fan-out to fast/accurate explicitly.
- **Forgetting to close the loop.** If the trace starts at a user, it should end back at that user. A diagram with a request going in and no response coming out is missing the most important arrow.
