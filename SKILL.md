---
name: system_diagram_in_mermaid
description: Draw system architecture diagrams in Mermaid with the discipline that makes them legible — trace-driven layout, semantic color, cross-cutting concerns on edges, surfaced architectural choices, and elk routing. Use this skill whenever the user asks for a system diagram, architecture diagram, request flow, data flow, service map, agent pipeline, or any "show me how X works at the system level" — even when they don't explicitly say "Mermaid." Also use when iterating on an existing diagram that the user thinks is messy, cluttered, or unclear. This skill replaces the default "draw boxes and arrows for every component you see" behavior with a procedural loop that produces diagrams a stranger can read.
---

# System diagram in Mermaid

A skill for drawing system architecture diagrams that are actually legible, using Mermaid as the rendering target and elk as the layout engine.

## Why this skill exists

The default failure mode for system diagrams is drawing the *noun inventory* of a system — every component as a box, every relationship as an arrow, color used decoratively, layers stacked vertically as if hierarchy were the same as call order. The result reads like an org chart of the codebase, not like an explanation of what the system does.

This skill encodes a different procedure: pick the right diagram for the job, plan the trace before drawing, encode meaning in color and edge style, surface the architectural choices that matter, and fence what's out of scope. Mermaid with elk routing solves the layout problem so the author doesn't have to.

## When to use this skill

Trigger on requests like:
- "Draw a diagram of [system]"
- "Show me how [X] works" where X is a multi-component system
- "Architecture of [project]" or "request flow for [feature]"
- "What does [agent / pipeline / service mesh] look like end to end"
- "Help me explain [system] to a new engineer / to compliance / on a slide"
- "This diagram is messy, can you redraw it" (iteration mode — see Section 5)

Do NOT use this skill for:
- ERDs / database schemas → use Mermaid `erDiagram` directly, no trace needed
- Sequence diagrams of pure message exchange → use Mermaid `sequenceDiagram` directly
- State machines → use Mermaid `stateDiagram-v2` directly
- Decision trees / flowcharts of pure logic (no system components) → use a plain flowchart
- UI mockups or visual designs → different problem entirely

The discriminator is whether the diagram needs to show **components of a system and how a request, signal, or piece of data moves through them.** That's what this skill is for.

## The procedure

Follow these six steps in order. Don't skip the planning steps and jump to Mermaid — that's how you get the failures this skill exists to prevent. Step 6 runs a panel critique on the diagram before returning to the user; it's a backstop, not a replacement for the procedure.

### 1. Pick the diagram's job

Before drawing anything, decide which of these the diagram is for:

- **Trace** — show one request flowing end to end through the system. (Default. Almost always the right choice for "draw a diagram of X.")
- **Topology** — show what components exist and what's connected to what, without a specific request flow. Use only when the user explicitly wants a map, not a story.
- **Failure / recovery path** — show what happens when something goes wrong. Always a separate diagram from the happy-path trace.
- **Cross-cadence loop** — batch processes, retraining loops, audit aggregation. Different time-scale than the request flow. Always a separate diagram.

If the user's request implies more than one of these (it usually does), draw the trace first and offer the others as follow-ups. Cramming multiple cadences into one diagram is the single most common failure mode.

State the choice in one line at the top of your response so the user can correct course before you commit to pixels: *"I'll draw this as a request trace — one user query flowing from the API gateway through the agent runtime to the audit log and back. The cross-run feedback loop and the failure path are different diagrams; I'll mention them at the end."*

### 2. Plan the trace in plain text first

Before writing Mermaid, write a short plain-text outline. This catches structural mistakes when they're cheap. The outline has four parts:

1. **Concrete entry point.** Pick a specific, named request — not "a user request" but something concrete like "user submits: 'summarize this week's incidents'" or "checkout: cart of 3 items, signed-in customer, default address." Specificity forces real decisions about what gets drawn.
2. **The path.** List the components the trace touches, in order, as a numbered list. If a component isn't on the path, it doesn't go on the diagram.
3. **The semantic axis.** State what color will encode. Pick exactly one: trust boundary, control plane vs data plane, sync vs async, internal vs external, or category (e.g. "infra / app / data"). Two axes confuse the reader; zero axes makes color decorative.
4. **Out of scope.** Name the things you're deliberately not drawing: failure paths, batch jobs, observability, etc. This is the fence that keeps the diagram from sprawling.

Show this outline to the user and ask for a quick sanity check before drawing. Five seconds of correction now saves a re-roll later.

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

```
flowchart LR
  %% ...
```

with init config:
```javascript
mermaid.initialize({
  flowchart: { defaultRenderer: 'elk', curve: 'basis', nodeSpacing: 50, rankSpacing: 60 },
});
```

If the user is rendering in an environment that defaults to dagre (GitHub, plain Markdown previews), warn them and either suggest Mermaid Live Editor with elk enabled, or accept the dagre rendering with a note that routing will be slightly worse.

The direction matters. `flowchart LR` (left-to-right) reads naturally for request traces. `flowchart TB` (top-to-bottom) is for hierarchies and stacks. Don't mix — pick one and stick to it.

### 5. If iterating on an existing messy diagram

When the user shows you a v1 / v2 diagram and says "make this better," don't just redraw — diagnose first. Read `references/diagnostic-checklist.md`, identify which specific failure modes the existing diagram exhibits, and tell the user which ones you're going to fix. Then go back to step 1.

The diagnostic mode is also useful when reviewing someone else's diagram without redrawing.

### 6. Panel critique and one revision round

Before returning to the user, run a one-round panel critique on the diagram. This catches the subtler failure modes the procedure can let through — out-of-scope sprawl, trust-axis violations, buried architectural choices, plan-violation drift.

Spawn **two** subagents in parallel. Each reads `references/panel-prompt.md` and applies the four-panelist critique procedure to your diagram independently, in its own context. Independent context matters — do not run the panel in your own context, since you'll grade work you just authored.

When both subagents return, **union their flagged issues**: collapse pairs that share both anchor and quoted span (one survives), keep all distinct issues. The unioned set is what feeds the partitioning step below. Two parallel runs catch stochastic flag drops — stage-1 evidence showed real issues catchable by one run can be missed by another with the same prompt against the same diagram. Wall-time cost is roughly one run (parallel); token cost roughly doubles for the panel step.

Partition the panel's flagged issues into two buckets:

**Revision-worthy** (revise once before returning to the user):

- Trace closure: `no-trace`, `missing-return-arrow`, `dead-end-nodes`
- Cadence and scope: `multiple-cadences`, `out-of-scope-sprawl`, `cadence-unnamed`
- Trust-axis violations: `wrong-trust-surface`, `tenant-shared-confused`
- Iteration-mode failures: `no-diagnosis`, `regression`
- Skill should have stepped aside: `wrong-sublanguage`, `forced-trace`
- Cross-cutter as flow participant: `cross-cutting-as-peer`
- ML serving leak in batch diagram: `serving-leak`

**Borderline** (surface in the panel summary; do not revise): everything else, including `noun-inventory` (any band), `choices-buried`, `decorative-color`, `unlabeled-cross-boundary`, `inconsistent-edge-style`, `gates-invisible`, `wrong-stage-order`. The user is in a better position than a reviser to judge whether these warrant changes.

If revision-worthy issues exist, revise the diagram once. The revision is **bounded to one round** — do not loop. When revising, do not consolidate or remove any node that the plan named as a load-bearing architectural choice, even to address a related anchor. Calibration found that count-driven or shape-driven consolidation tends to bury structural choices the plan called out as load-bearing — the plan's stated intent takes precedence.

If every panelist returned `verdict: ship`, skip the revision and surface a one-line "panel clean" note in the panel summary.

The full design rationale, the calibration findings that produced this filter, and open questions are in `design/integrated-flow-sketch.md` and `design/refinement-loop.md`.

## Output format

Return four things:

1. **The plan** (from step 2) as a short prose block, so the user can verify the trace and scope.
2. **The Mermaid source** in a fenced code block, ready to paste into a Markdown file or Mermaid Live Editor — post-revision if step 6 revised it.
3. **A brief note** about (a) what's out of scope and where it would go in a sibling diagram, (b) any architectural choices that are surfaced visually, and (c) the renderer config the diagram assumes.
4. **A panel summary**, at most three sentences:
   - If panel-clean: name the four panelists and that all returned `ship`. Example: *"Panel clean — Trace Reader, Visual Encoding Critic, Scope Steward, and the SME (Backend / API engineer) all returned ship."*
   - If revised: name the issues addressed (anchor + one-line description), then name the borderline issues surfaced (anchor + one-line description), if any. Example: *"Panel revised one issue (`out-of-scope-sprawl` — the offline pipeline was drawn despite being declared out of scope). Two borderline issues surfaced: `inconsistent-edge-style` on the FAISS/sklearn branch, and `noun-inventory` at 14 nodes (soft band)."*

   Do not surface the full panel JSON unless the user asks for it. Verbose surfacing defeats the point of the borderline classification.

If the user asks for the diagram to render inline (e.g. in a Claude artifact, in chat, in a doc with live Mermaid support), provide an HTML wrapper that loads Mermaid v11 from esm.sh, configures elk, and renders the source. Pattern is in `references/mermaid-patterns.md` under "Standalone HTML render."

## Common failure modes to avoid

Even with this procedure, watch for these tendencies:

- **Drawing the noun inventory.** If your diagram has more than ~12 nodes or every component from the source material appears as a box, you've slipped into inventory mode. Cut to the trace.
- **Layer cake.** Three or more horizontal bands stacked vertically with arrows mostly going down is the layer-cake antipattern. A plan-execute-verify loop, an audit sink, and a cross-run feedback channel can't be drawn as a stack — they're loops and side-channels. Use subgraphs and explicit return arrows instead.
- **Decorative color.** If your color choices don't map to the semantic axis you named in step 2, they're decorative. Either fix the mapping or go monochrome.
- **Cross-cutting concerns as boxes.** Auth, PII redaction, rate limiting, audit logging, observability — these intercept the flow, they don't participate in it. They go on edges as labels or as gates, not as peer nodes.
- **Architectural choices buried in subtitles.** "LLM Router (cost-aware tiers)" hides the interesting bit. If tier selection matters, draw the fan-out to fast/accurate explicitly.
- **Forgetting to close the loop.** If the trace starts at a user, it should end back at that user. A diagram with a request going in and no response coming out is missing the most important arrow.

## Reference files

- `references/diagnostic-checklist.md` — failure modes for diagnosing an existing diagram before redrawing. Read this before iterating on user-provided diagrams.
- `references/mermaid-patterns.md` — specific Mermaid idioms: subgraph styling, classDef patterns, linkStyle indexing, elk config, and a standalone HTML render template. Read this when writing the Mermaid source if you're unsure about a specific construct.
- `references/panel-prompt.md` — the four-panelist critique procedure used by step 6. The skill spawns a subagent that loads this file and applies the panel to the diagram before returning to the user.
