# Reader-test prompt

A self-contained prompt for the Phase D reader test. Spawn a fresh subagent with no other context, hand it the produced architecture doc *and only the doc* (no source code, no repo access, no prior conversation), and ask it five archetypal onboarding questions. The subagent answers from the doc alone. If it can't, that's a doc gap — not a subagent gap.

This is the eval-driven analog of running an eval on a retrieval system: the doc has a job (onboard a stranger to the codebase); the reader test measures whether the doc actually does that job. A doc-panel critique catches structural and coherence issues; the reader test catches "the doc has all the right sections but somehow doesn't actually answer the questions a stranger would ask."

Used by `SKILL.md` Phase D. The reader test runs on every doc that hasn't opted out, including 3-section docs (the panel critique skips these; the reader test does not — short docs are precisely the ones at risk of "looks fine, doesn't onboard").

---

## How to use this file

1. Spawn a subagent with **no access to the source repo and no prior conversation context** — this is load-bearing. The subagent must answer from the doc alone, not from priors.
2. Paste everything below the `--- BEGIN PROMPT ---` marker into that subagent's prompt.
3. Beneath the prompt, paste:
   - The produced architecture doc (full markdown, including diagrams).
   - Five questions tailored to this specific doc (see "Generating the questions" below).
4. Read the JSON report the subagent returns.
5. Apply the verdict per the filter at the end of this file.

The subagent does not get the SKILL.md, the references, the source code, or any context about how the doc was generated. It plays the role of a new engineer reading the doc cold.

## Generating the questions

The skill (the *parent* of the subagent) generates the five questions before invoking the reader test, based on the produced doc:

1. **Where to start.** "If I'm new to this codebase, what file should I read first and why?" (Always this exact question — it tests the where-to-start section.)

2. **Component understanding.** "What does `<X>` do, and what does it deliberately *not* do?" Pick `<X>` as a load-bearing component named in the doc — typically the one with the most edges in the architecture-overview diagram, or the one mentioned earliest in the component-summaries section. Test of the component-summary template's surface/boundary discipline.

3. **Data flow.** "How does data move from `<input>` to `<output>` through the system?" Pick `<input>` and `<output>` as the boundaries of the headline trace (e.g., "PDF" → "top-k chunks", "user request" → "API response"). Test of the architecture-overview prose+diagram coherence.

4. **Persistence.** "What persists between runs of this system, and where does it live?" (Always this exact question — tests the state-and-persistence section, or the pieces of state-and-persistence folded into other sections.)

5. **Boundaries.** "What is deliberately out of scope for this system, and where would I look for that information instead?" (Always this exact question — tests the out-of-scope section.)

The first, fourth, and fifth questions are universal. The second and third are per-doc. The skill picks the targets for questions 2 and 3 based on what the doc named as load-bearing.

If the doc deliberately omitted the state-and-persistence section (with a stated reason), the skill substitutes question 4 with: "What's the system's relationship to persistent state, and is there any?"

---

--- BEGIN PROMPT ---

You are role-playing as a new engineer who has been pointed at a codebase and given an architecture doc to read. You have not seen the source code. You have not seen any other documentation. You are reading this doc cold.

Your job is to answer five questions from the doc alone. For each question, you return:

- `status`: one of `answered`, `partial`, or `cant-answer`.
  - `answered`: the doc gives you a clear, specific answer with concrete file pointers or named components or quoted phrases.
  - `partial`: the doc gestures at the answer but lacks the specifics you'd need to act on it. Examples: a component is named but its responsibility is described in vague terms; a flow is shown in a diagram but the prose doesn't explain the steps; a section title promises an answer but the section content doesn't deliver.
  - `cant-answer`: the doc has nothing to say on this. You cannot extract an answer with reasonable interpretation.
- `evidence_in_doc`: the specific section name, paragraph, file pointer, or diagram element you used to answer (or "(none)" if `cant-answer`). Quote when possible.
- `gap`: if `status` is `partial` or `cant-answer`, name what specifically is missing. One sentence.

Be honest. The reader test exists to find doc gaps, not to validate the doc. If the doc is shaped well but you couldn't extract a specific file pointer for question 1, that's `partial`, not `answered` — record what would have moved it from partial to answered.

You do **not** evaluate the prose quality, the diagram aesthetics, the section ordering, or any structural property. You evaluate only: "could a stranger answer these five questions from this doc alone."

## Output format

Return a single JSON object, no prose before or after, no markdown code fence around the JSON:

```json
{
  "verdict": "pass",
  "questions": [
    {
      "question": "If I'm new to this codebase, what file should I read first and why?",
      "status": "answered",
      "evidence_in_doc": "Where to start reading section, bullet 1: 'README.md — quick start, the six CLIs, and the headline eval numbers. The fastest on-ramp.'",
      "gap": null
    },
    {
      "question": "What does normalize.py do, and what does it deliberately not do?",
      "status": "partial",
      "evidence_in_doc": "Component summaries section, normalize.py subsection. Says it 'aggregates per-page parses into a single contract-aware document' and lists three passes.",
      "gap": "Boundary line is missing. The doc says what normalize.py does but doesn't name what it doesn't do — e.g., does it OCR? does it chunk? a stranger could reasonably guess wrong."
    }
  ],
  "doc_quality_summary": "The doc onboards well on architecture overview and where-to-start. Component summaries are inconsistent on the boundary discipline; question 2's normalize.py subsection lacked the explicit 'doesn't do' line."
}
```

Field semantics:

- `verdict`: `pass` if all five questions are `answered`; `revise` if any is `cant-answer`; `borderline` if some are `partial` and none is `cant-answer`.
- `questions`: exactly 5 entries, in the order asked.
- `doc_quality_summary`: one or two sentences. Specific. Names which questions struggled, not just "doc was good/bad."

If the doc is sound on all five:

```json
{
  "verdict": "pass",
  "questions": [
    {"question": "...", "status": "answered", "evidence_in_doc": "...", "gap": null},
    {"question": "...", "status": "answered", "evidence_in_doc": "...", "gap": null},
    {"question": "...", "status": "answered", "evidence_in_doc": "...", "gap": null},
    {"question": "...", "status": "answered", "evidence_in_doc": "...", "gap": null},
    {"question": "...", "status": "answered", "evidence_in_doc": "...", "gap": null}
  ],
  "doc_quality_summary": "All five onboarding questions answered with specific evidence. The doc reads coherently from where-to-start through out-of-scope."
}
```

--- END PROMPT ---

## Filter table for the caller

The skill applies the reader-test verdict using this filter:

| Verdict      | Action                                                                       |
| ------------ | ---------------------------------------------------------------------------- |
| `pass`       | Ship. Surface "reader test: 5/5 answered" in generation notes.               |
| `borderline` | Ship. Surface the partial answers and their gaps in generation notes; do not revise. |
| `revise`     | Revise once, bounded. Regenerate the section(s) corresponding to the failed question(s). After revision, do not re-run the reader test — bounded to one round. Surface the original gaps and what was changed in generation notes. |

The mapping from question to section:

- Question 1 → Where to start reading
- Question 2 → Component summaries (or whichever section defined `<X>`)
- Question 3 → Architecture overview (the headline trace)
- Question 4 → State and persistence (or wherever persistence is described, if folded)
- Question 5 → Out of scope

A `cant-answer` on question 4 with no state-and-persistence section in the doc is *not* a revision trigger if the doc plan deliberately omitted state-and-persistence with a stated reason. In that case, the gap is acceptable; surface in generation notes as "deliberately out of scope per doc plan; reader test confirms."

## When to skip

The reader test is required by default. Skip only when:

- The user has explicitly opted out of all critique loops via Phase A signals ("rough overview", "skip the design pass", "no critique").
- The skill is being invoked in iteration mode on an existing diagram (`references/diagnostic-checklist.md`), not on a full doc.

Phase 0's in-session-author detection does *not* skip the reader test, even though it skips the panel critiques. The reader test runs in a fresh subagent with no source-code context, so the in-context self-grading concern doesn't apply — that's the whole point of the design.

## Why this exists

A doc that passes the doc-panel critique can still fail to onboard a stranger. The panel checks structural anchors (sections present? prose grounded? out-of-scope named?) but doesn't simulate the stranger's experience of *trying to use the doc to answer real questions*. The reader test is the missing yardstick — the same way a retrieval eval set is the missing yardstick for a retrieval pipeline. Passing the panel says "the doc has the right shape." Passing the reader test says "a stranger can use the doc."

Both matter; neither replaces the other.
