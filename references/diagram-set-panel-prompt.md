# Diagram-set panel critique prompt

A self-contained prompt for critiquing a **diagram set design** (the output of step 0 of `references/diagram-procedure.md`, specified in `references/diagram-set-design-pass.md`) before any Mermaid is generated. Paste this file as context, paste the set design beneath it, and Claude returns a JSON report on whether the set composition is sound.

This is distinct from `references/panel-prompt.md` (per-diagram critique) and `references/doc-panel-prompt.md` (doc-level critique). The diagram-set panel reviews the *set composition within a doc section*: aspect coverage, cadence splits, zoom anchoring, fan-out vs the system's actual complexity. Used by the inner diagram procedure only when the set has more than one diagram (single-diagram sets don't need a set-level critique).

---

## How to use this file

1. Paste everything below the `--- BEGIN PROMPT ---` marker into a fresh Claude Code session.
2. Beneath the prompt, paste the diagram set design (the prose block from step 0) plus the user's original request.
3. Read the JSON report Claude returns.
4. Apply revision-worthy issues by adjusting the set design, then re-running step 0 if the changes were substantial.

---

--- BEGIN PROMPT ---

You are running a single-panelist critique on a diagram set design. The design names what diagrams will be drawn for a system, their archetypes, scope, and relationships. Your job is to catch composition-level failures before any Mermaid is generated. You are NOT critiquing individual diagrams — those don't exist yet.

Read the set design carefully. Compare it against the user's original request and what the system appears to be. Then evaluate against the rubric below.

## Rubric

For each anchor, decide whether it fires. If it fires, return an issue with the set element it applies to (or "set-level" if it's a property of the whole set), one sentence of explanation, and a recommended fix.

**Coverage anchors:**

- `missing-aspect` — a major aspect of the system is absent from the set when the system clearly has it. Common cases: system has cross-run feedback / learning loop but no cadence sibling for it; system has explicit failure / retry semantics but no failure sibling; system has load-bearing internal structure (multi-stage component) but no zoom sibling. Anchor fires when the user's request or system description mentions an aspect that no diagram in the set covers.
- `under-fanout` — the system has multiple cadences, multi-loop structure, or a load-bearing zoomable component, but the set has only 1 diagram. Anchor fires when the headline diagram is going to sprawl trying to cover everything.

**Composition anchors:**

- `wrong-cadence-split` — sync and async aspects are in the same diagram when they should be siblings, OR an artificial split was introduced when one cadence would suffice. Anchor fires when the cadence boundaries in the design don't match the system's actual time scales.
- `redundant-siblings` — two siblings in the set cover overlapping ground; one should be cut or merged. Anchor fires when two diagrams have ≥60% scope overlap in the elements they would draw.
- `over-fanout` — the system is simple but the set has 4–5 diagrams; reduce. Anchor fires when at least one sibling is below the "would only have 3–4 nodes" threshold and should be folded into NOTES.

**Anchoring anchors:**

- `headline-unclear` — the set has no obvious "read this first" diagram. Anchor fires when no diagram is marked `[headline]` or when multiple diagrams compete for that role without a defined relationship.
- `unanchored-zoom` — a "zoom" sibling doesn't tie back to a specific component in the headline. Anchor fires when a zoom diagram's parent component isn't named in the headline diagram's scope.

## What NOT to flag

- The specific scope of each diagram — that's step 2's job (per-diagram planning), not yours.
- Naming, color choices, render config — all per-diagram concerns.
- "I would have drawn it differently" — only flag concrete failures from the rubric, not preferences.
- Single-diagram sets — if the design has only one diagram, the design-panel shouldn't have been invoked. Return `all_clear: true` and note the set size in `notes`.

## Output format

Return a single JSON object, no prose before or after, no markdown code fence around the JSON:

```json
{
  "set_size": 4,
  "verdict": "revise",
  "issues": [
    {
      "anchor": "missing-aspect",
      "applies_to": "set-level",
      "explanation": "The system description mentions a cross-run feedback loop (verdicts feeding LearningEngine) but no diagram in the set covers it; this is a distinct cadence that won't fit on the headline trace.",
      "recommended_fix": "Add a cross-cadence sibling for the feedback loop (verdicts → LearningEngine → next-run prompts)."
    },
    {
      "anchor": "unanchored-zoom",
      "applies_to": "diagram 2",
      "explanation": "Diagram 2 zooms 'Phase 2 deep dive' but Phase 2 is not named as a node in diagram 1's scope.",
      "recommended_fix": "Either rename a node in diagram 1 to 'Phase 2 deep dive' so the zoom has a parent, or remove the zoom sibling."
    }
  ],
  "notes": "Set is well-fanned out at 4 diagrams; the missing feedback-loop cadence is the load-bearing gap. Other anchors clean."
}
```

Field semantics:

- `set_size`: integer count of diagrams in the design.
- `verdict`: `"ship"` if no issues fire, `"revise"` otherwise.
- `issues`: zero or more, each with `anchor` (from the rubric above), `applies_to` (a diagram number like `"diagram 2"` or `"set-level"`), `explanation` (one sentence), and `recommended_fix` (one sentence — concrete, actionable).
- `notes`: one or two sentences for the human reader. Keep terse.

If the set is sound:

```json
{"set_size": 1, "verdict": "ship", "issues": [], "notes": "Single-diagram set; design-panel critique is not the right tool here."}
```

or

```json
{"set_size": 3, "verdict": "ship", "issues": [], "notes": "Set composition is sound; cadence boundaries align with the system's actual time scales and the zoom is properly anchored."}
```

--- END PROMPT ---
