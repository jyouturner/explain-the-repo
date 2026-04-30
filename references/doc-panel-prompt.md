# Doc-panel critique prompt

A self-contained prompt for critiquing an architecture doc — either at the *plan* level (Phase A) before generation, or at the *assembled* level (Phase C) after generation. Paste this file as context, paste the doc plan or full doc beneath it, and Claude returns a JSON report on coherence and completeness.

This is distinct from `references/panel-prompt.md` (per-diagram critique) and `references/diagram-set-panel-prompt.md` (within-section diagram-set critique). The doc-panel reviews the *whole-doc composition*: section coverage, section ordering, prose-diagram coherence, pointer resolution, grounding in actual code.

Used by `SKILL.md` Phase A (plan-level review when the plan has more than 3 sections) and Phase C (assembled-doc review). Skipped for 3-or-fewer-section docs and opt-out runs.

---

## How to use this file

1. Paste everything below the `--- BEGIN PROMPT ---` marker into a fresh Claude Code session.
2. Beneath the prompt:
   - **Phase A (plan review):** paste the doc plan (the prose block from `references/doc-design-pass.md`).
   - **Phase C (assembled doc review):** paste the full assembled markdown doc.
   Indicate which mode you're running.
3. Read the JSON report Claude returns.
4. Apply revision-worthy issues per the filter table below.

---

--- BEGIN PROMPT ---

You are running a single-panelist critique on an architecture doc. The doc is meant to explain a codebase to a stranger — a single markdown file, 300–800 lines, 3–10 sections, combining prose and Mermaid diagram sets. Your job is to catch composition and coherence failures.

You will be told whether you are reviewing a **plan** (Phase A — section list before generation) or an **assembled doc** (Phase C — generated markdown after per-section work). The rubric below has anchors that apply to one or both modes.

You are NOT critiquing individual diagrams or per-section internals — those have their own panels. You are reviewing *the doc as a whole*.

## Rubric

For each anchor, decide whether it fires. If it fires, return an issue with the section it applies to (or "doc-level"), one sentence of explanation, and a recommended fix.

**Coverage anchors (apply to plan and assembled doc):**

- `missing-section` — the system has a major aspect that no section covers. Common cases: system has nontrivial persistent state but no state-and-persistence section; system has a clear build-time/run-time split but no lifecycle section; system has heavy domain jargon but no glossary; doc has no `where-to-start` section (which is non-optional).
- `over-sectioned` — the doc has more than 8 sections AND at least one section is below the "3 paragraphs of unique content" threshold. Recommend folding or cutting.
- `under-sectioned` — the doc has 3 sections and the system clearly has multi-component complexity, persistent state, or lifecycle distinctions that aren't covered.

**Composition anchors (apply to plan and assembled doc):**

- `wrong-section-order` — sections in an order that confuses a first-time reader. Canonical examples: failure modes / runbook before architecture overview; out-of-scope at the top; glossary in the middle (it should be at or near the end).
- `redundant-sections` — two sections cover overlapping ground (≥ 60% overlap of source files referenced or concepts described); one should be merged or cut.

**Coherence anchors (apply to assembled doc only):**

- `prose-diagram-mismatch` — the prose says one thing about a component, the adjacent diagram shows another. Concrete forms: prose says "service A calls service B"; diagram has no edge from A to B. Or: diagram has node X labeled "cache"; prose calls it "buffer". Quote the conflicting sentence and the diagram element.
- `unanchored-pointers` — the doc says "see the X service" or "the Y handler" but X / Y is never defined or pointed at. Quote the sentence with the dangling reference.
- `not-grounded` — the prose makes specific claims (about call structure, file paths, function names, persistence) that aren't backed by file references in the doc. Quote the unsourced claim.
- `where-to-start-missing-or-thin` — the `Where to start reading` section is absent OR has fewer than 3 file pointers. This section is non-optional and load-bearing.

**Anchoring anchors (apply to plan and assembled doc):**

- `no-out-of-scope-section` — the doc lacks an `Out of scope` section at the end naming what it doesn't cover. Even a one-line acknowledgment is required.
- `headline-overreaches` — the headline paragraph promises content that the doc doesn't actually cover (e.g., headline says "this doc covers deployment and operations" but no such section exists).

## What NOT to flag

- Per-diagram issues (color, layout, missing return arrows on a single diagram). Those are the diagram-level panel's job.
- Within-section prose quality (verbose, awkward phrasing, etc.). Surface only structural / composition issues.
- "I would have organized it differently" — only flag concrete failures from the rubric, not preferences.
- Section content depth — if a section is shorter than expected but has all the load-bearing details, that's fine.
- Missing sections that the doc plan deliberately omitted with a stated reason. Trust the omission unless the reason looks weak.

## Output format

Return a single JSON object, no prose before or after, no markdown code fence around the JSON:

```json
{
  "mode": "plan",
  "section_count": 6,
  "verdict": "revise",
  "issues": [
    {
      "anchor": "missing-section",
      "applies_to": "doc-level",
      "explanation": "The system has cross-run feedback state (FeedbackRegistry, LearningEngine) but no state-and-persistence section covers it; it would only show up implicitly inside the architecture overview's diagrams.",
      "recommended_fix": "Add a state-and-persistence section between architecture-overview and out-of-scope, grounding in src/registries.py and src/learning.py."
    },
    {
      "anchor": "wrong-section-order",
      "applies_to": "section 5 (Glossary)",
      "explanation": "Glossary is currently between component-summaries and architecture-overview; it should be at or near the end of the doc, after the substantive sections.",
      "recommended_fix": "Move Glossary to the second-to-last position, before Out of scope."
    }
  ],
  "notes": "Section coverage is mostly good; the missing state-and-persistence section is the load-bearing gap. Other sections are correctly classified and grounded."
}
```

Field semantics:

- `mode`: `"plan"` if reviewing a doc plan (Phase A), `"assembled"` if reviewing the assembled markdown (Phase C).
- `section_count`: integer count of sections in the plan or doc.
- `verdict`: `"ship"` if no issues fire, `"revise"` otherwise.
- `issues`: zero or more, each with `anchor`, `applies_to` (a section name like `"section 4 (Component summaries)"` or `"doc-level"`), `explanation` (one sentence), `recommended_fix` (one sentence — concrete and actionable).
- `notes`: one or two sentences summarizing the doc's overall state. Keep terse.

## Filter table for the caller

The skill applies the panel's verdict using this filter:

| Anchor                            | Bucket          |
| --------------------------------- | --------------- |
| `missing-section`                 | revision-worthy |
| `wrong-section-order`             | revision-worthy |
| `prose-diagram-mismatch`          | revision-worthy |
| `unanchored-pointers`             | revision-worthy |
| `not-grounded`                    | revision-worthy |
| `where-to-start-missing-or-thin`  | revision-worthy |
| `no-out-of-scope-section`         | revision-worthy |
| `headline-overreaches`            | revision-worthy |
| `over-sectioned`                  | borderline      |
| `under-sectioned`                 | borderline      |
| `redundant-sections`              | borderline      |

Coverage and coherence anchors are revision-worthy because the failure mode is "doc is incomplete or contradictory." Sectioning judgment calls are borderline — surface to the user, who knows what they want better than the reviser does.

If the assembled doc has only revision-worthy issues that affect specific sections (not doc-level), regenerate only those sections; don't redo the whole doc.

## When the doc is sound

```json
{"mode": "assembled", "section_count": 5, "verdict": "ship", "issues": [], "notes": "Sections cover the load-bearing aspects; prose grounds in cited files; diagrams and prose agree; where-to-start has 4 specific pointers; out-of-scope present."}
```

--- END PROMPT ---
