# Diagram-set design pass — picking the set within a doc section

The diagram procedure produces a **diagram set** for a section, not a single diagram. This file is the procedure for deciding what's in the set before any drawing begins. The default set size is 1; you add diagrams when the system aspect being illustrated has properties that genuinely don't fit one frame.

This is step 0 of `references/diagram-procedure.md` — the inner diagram loop. Runs before steps 1–6 of that procedure, which then run *per diagram in the set*. Distinct from the doc-level design pass (`references/doc-design-pass.md`), which decides which doc *sections* exist.

---

## What a diagram set is

A set of 1–5 sibling diagrams that together cover what the user wants to understand about the system. Each diagram is clean within its own scope (~6–12 nodes). The set has explicit relationships: which diagram is the headline, which zooms a component in another, which is a cadence sibling, and so on.

Crucially: a set of 5 small clean diagrams is almost always more legible than 1 sprawling 22-node diagram trying to do everything.

## When to add a sibling

You start with 1 diagram (the headline trace). Add siblings only when you can name a specific reason from this list:

1. **Cadence sibling.** The system has flows that run on different time scales — sync request (ms) + async batch (minutes) + cross-run feedback (days). Mixing cadences in one frame is the canonical failure mode named in `references/diagnostic-checklist.md` § Multiple cadences in one frame. Each cadence becomes its own trace.

2. **Failure sibling.** The failure / recovery path is structurally distinct from the happy path. Drawing both in one frame creates dual flows that compete for the reader's attention. The failure path becomes its own trace.

3. **Zoom sibling.** A specific component on the headline trace has internal structure load-bearing enough to deserve its own diagram (Phase 2 deep-dive sub-pipeline in an agent; resolver fan-out in an API; trainer phases in an ML pipeline). The parent is the headline; the zoom explodes one node into its own trace.

4. **Topology sibling.** State stores, deployment topology, or component dependencies don't naturally appear in a request trace. A static topology diagram answers "what exists and how is it connected"; a trace answers "how does a request flow." Different questions, different diagrams.

5. **Lifecycle sibling.** Systems with distinct build-time and run-time concerns (training + serving, build pipeline + runtime) may deserve a sibling per lifecycle stage with the artifact handoff as the boundary. (Two-tower ML serving is the canonical case.)

If none of these reasons applies, keep the set at 1.

## When to stop adding

1. **Cap at ~3–5.** More than 5 stops being a diagram set and becomes a documentation suite. If the system genuinely needs 7 diagrams, the right answer is a docs overhaul, not a one-shot skill response.

2. **Don't add topology when the trace already names everything.** If the headline trace already mentions every component the user cares about, a separate "components" diagram is redundant.

3. **Sub-3-node siblings belong in NOTES.** If a "sibling" would have only 3–4 nodes, mention it in the headline diagram's NOTES instead of drawing it. Drawing a 3-node diagram is more overhead than information.

4. **Don't split for symmetry.** If you've drawn one detailed sibling, you don't *also* need to draw a sibling for every other component. Add only siblings that earn it via the rules above.

## Rough decision rule

- Start at 1.
- +1 for each genuinely different cadence (max +2 — three cadences in one set is rare and probably means rescope).
- +1 for at most one zoom-in of the most load-bearing component.
- +1 for a topology / lifecycle sibling **if** it answers a question the trace can't.
- Cap at 5.

Most systems land at 1–3. The autoresearch-shaped agents land at 4–5 because they're genuinely multi-cadence with load-bearing internal structure. The graphql-node-shaped APIs land at 1.

## Format for the set design

Produce this as a short prose block before generating any Mermaid:

```
DIAGRAM SET — <one-line system description>

Composition (headline + siblings):
1. [headline] <archetype: trace|topology|failure|cross-cadence>: <one-line scope>
   Concrete entry point or representative element: <what>
2. [sibling, <relationship>] <archetype>: <one-line scope>
   Concrete entry point or representative element: <what>
3. ...

Relationships:
- Diagram 2 zooms component <X> within Diagram 1.
- Diagram 3 is a cadence sibling (async vs Diagram 1's sync).
- Diagram 4 is a topology view answering <question> that the traces don't.

Aspects deliberately out of the set:
- <aspect>: <reason>

Estimated set size: <N> diagrams. Reasoning: <one sentence — which sibling rules applied>.
```

The relationships section is load-bearing. Don't draw a sibling without naming how it relates to another diagram in the set. Floating siblings that nobody can locate are a design failure mode.

## Surface to the user before drawing

Show the set design to the user and ask for a quick sanity check before any generation begins. Five seconds of correction at the design level prevents minutes of regeneration after.

A typical surface:

> "I'll draw this as a 4-diagram set: a request trace as the headline, a Phase-2 zoom that explodes the orchestrated-vs-single-call pipeline, a cross-run feedback cadence sibling, and a state-stores topology sibling. The dashboard observability is out of the set entirely. Sound right before I draw?"

If the user dismisses the design pass ("just give me one diagram"), respect that — drop to N=1 and use the headline only.

## When the design pass is over

You have:
- A set of N diagrams (1 ≤ N ≤ 5).
- Each with an archetype, scope, and concrete entry point or representative element.
- Named relationships between them.
- A list of aspects deliberately excluded from the set.

For each diagram in the set, run steps 1–6 of the main procedure. Step 1 ("pick the diagram's job") is now narrowed by the set design — the archetype is fixed; step 1 confirms the specific job.

The set design itself is reviewed by a separate subagent — see `references/diagram-set-panel-prompt.md`. That review runs only when N > 1 (single-diagram sets don't need a set-level critique; the existing diagram-panel covers them).
