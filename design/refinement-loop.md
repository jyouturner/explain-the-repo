# Refinement loop design

Status: draft, pre-implementation. This document proposes a generate-critique-revise loop ("refinement loop") layered on top of the existing five-step procedure in `SKILL.md`. Nothing here is implemented yet.

## Goal

Today the skill produces one diagram per request. "Beautiful" is operationalized through `references/diagnostic-checklist.md` and `examples/anti-patterns.md`, but those checks happen only when the user *complains* and asks for an iteration (Section 5 of `SKILL.md`). We want the same checks to run automatically on the skill's own first draft, so that what reaches the user is already a third draft critiqued through several distinct lenses — without the user having to play diagnostician.

The skill is markdown-native today (no Python, no runtime deps), and we want to keep it that way. So this design stays inside the skill ecosystem rather than building an external eval framework.

Two stages, gated:

1. **Stage 1 — `references/panel-prompt.md`.** A single markdown file containing the panel-critique prompt. Hand-run by pasting a diagram into a Claude Code session with this file as context. Goal: cheaply answer "does the panel catch failures the single-shot skill misses?" before investing in any automation. If the answer is no, we stop here.
2. **Stage 2 — a slash command (only if stage 1 earns it).** A `.claude/commands/` markdown command that drives the panel automatically — either against the previous diagram in the current conversation (a `/refine`-shaped UX) or against the prompts in `evals/evals.json`. Still markdown-native, no Python.

Explicitly *not* a deliverable: any Python harness, custom runner, or external eval framework. If stage 2 needs more than a slash command can express, that's a signal to revisit, not to add a new toolchain.

## Why a panel, not a single judge

LLM-as-judge with a generic "is this a good diagram?" prompt collapses onto whatever rubric is most heavily represented in the model's fine-tuning data — usually a vague mix of "clear," "complete," and "well-organized." That single judge cannot distinguish between *a layer cake that looks tidy* and *a trace that looks scrappy but actually communicates*, because both score "okay" on a generic rubric.

The panel approach (Karpathy's "channel many perspectives" framing) trades the single generic rubric for several disjoint specific ones. Each panelist owns a non-overlapping slice of the failure-mode taxonomy we already wrote down. The panel only converges to "ship" when every slice clears, which forces the model to address each failure mode separately rather than averaging them away.

The panel does not solve subjectivity. It makes subjectivity negotiable: instead of one fuzzy score, you get four crisp verdicts plus the specific issues each panelist flagged.

## The loop

```
                ┌─────────────────────────────────┐
                │  1. Generate                    │
                │  Run the existing 5-step        │
                │  procedure from SKILL.md.       │
                │  Output: plan + Mermaid source. │
                └─────────────────────────────────┘
                                │
                                ▼
                ┌─────────────────────────────────┐
                │  2. Detect archetype            │
                │  Map prompt to one of the 5     │
                │  evals.json archetypes (or      │
                │  "other"). Selects the SME.     │
                └─────────────────────────────────┘
                                │
                                ▼
                ┌─────────────────────────────────┐
                │  3. Panel critique (subagent)   │
                │  Single subagent simulates all  │
                │  4 panelists. Each returns:     │
                │    - verdict: ship | revise     │
                │    - 0–3 specific issues        │
                │  Disjoint rubrics; see below.   │
                └─────────────────────────────────┘
                                │
                                ▼
                ┌─────────────────────────────────┐
                │  4. Stop?                       │
                │  - all panelists "ship" → done  │
                │  - budget exhausted → done      │
                │  - issues unchanged from last   │
                │    round (oscillation) → done   │
                │  - otherwise → revise           │
                └─────────────────────────────────┘
                                │
                                ▼
                ┌─────────────────────────────────┐
                │  5. Revise                      │
                │  Feed plan + Mermaid + flagged  │
                │  issues back to the generator;  │
                │  produce a new draft. Loop.     │
                └─────────────────────────────────┘
```

Step 1 is unchanged from today. Steps 2-5 are new.

## The panel

Four panelists. Each owns a disjoint slice of `references/diagnostic-checklist.md` and `examples/anti-patterns.md`. Disjointness is enforced by giving each panelist explicit "not your job" guidance — a Trace Reader does not get to complain about color choices.

### Panelist 1 — Trace Reader

**Lens:** "Can a stranger follow this end to end without prior context?"

**Rubric, anchored to `diagnostic-checklist.md`:**
- *No trace.* Is there a clear entry point and exit point? Can I name them?
- *Missing return arrow.* Does the request close back to the originator?
- *Dead-end nodes.* Any boxes with no outgoing edge that aren't sinks?
- *Multiple cadences in one frame.* Is more than one time-scale being drawn?

**Out of scope for this panelist:** color, label placement, choice of subgraph axis, scope/inventory.

### Panelist 2 — Visual Encoding Critic

**Lens:** "Does the visual encoding carry information, or is it decoration?"

**Rubric, anchored to `diagnostic-checklist.md` § Semantic problems and `anti-patterns.md` § Decorative color, § Layer cake:**
- *Decorative color.* Can the named semantic axis be inferred from color alone? If two same-color nodes don't share a category along that axis, fail.
- *Layer cake.* Are three or more horizontal bands stacked vertically with arrows mostly going down?
- *Unlabeled cross-boundary edges.* Does every edge that leaves a subgraph carry a label naming what flows on it?
- *Inconsistent edge style.* Are dotted edges used consistently for side effects (audit, lookup, async) and solid edges for the main trace?

**Out of scope:** trace correctness, scope, domain accuracy.

### Panelist 3 — Scope Steward

**Lens:** "What's on the diagram that shouldn't be, and what's missing that the trace requires?"

**Rubric, anchored to `anti-patterns.md` § Noun inventory, § Cross-cutting concerns drawn as peer boxes, § Architectural choices buried in subtitles:**
- *Noun inventory.* More than ~12 nodes, or every component from the prompt appears with equal weight?
- *Cross-cutting concerns as peers.* Auth, PII redaction, audit, rate limiting, observability drawn as nodes with their own arrows instead of as edge labels or sinks?
- *Architectural choices buried.* Are decisions like cost-aware routing, rules-then-LLM verification, or lazy loading hidden in parenthetical text instead of drawn structurally?
- *Out-of-scope sprawl.* Is the diagram trying to also be the failure-path / batch-loop / observability diagram?

**Out of scope:** color, layout, render config.

### Panelist 4 — Domain SME (variable)

**Lens:** "Does this look right for *this kind of system*?"

The SME persona is selected by the archetype detected in step 2. This is the panelist that prevents the other three from converging to a generic "well-formatted diagram" verdict — only the SME can say "your RAG diagram is missing the retriever-vs-generator split that anyone in this domain would expect."

| Archetype (from `evals.json`)             | SME persona                | What they specifically check                                                                  |
| ----------------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------- |
| `request trace with trust boundaries`     | Platform / SRE engineer    | Trust boundaries match real isolation surfaces; tenant-scoped vs shared is correctly drawn.   |
| `request trace, no trust boundaries`      | Backend / API engineer     | Stages map to real CI/API stages; gates (manual approval, green tests) are visible.           |
| `cross-cadence loop` (e.g. ML pipeline)   | ML / data engineer         | Cadence named explicitly; storage components drawn as cylinders; no synchronous serving leak. |
| `diagnostic mode on existing diagram`     | Senior diagram reviewer    | Diagnosis precedes redraw; 3-4 named failure modes; redraw doesn't reproduce them.            |
| `skill should NOT trigger` (ERD, etc.)    | Mermaid generalist         | Skill stepped aside; correct Mermaid sub-language was used.                                   |
| *fallback: archetype not detected*        | "Curious new hire"         | Can I tell what this system does? What's the one question I'd ask after reading it?           |

The SME's rubric is intentionally smaller (2-3 items) and more domain-specific than the other three. The SME has explicit license to flag things the rubric didn't anticipate, as long as they're domain-specific (not "this should be more colorful").

### Anti-collapse measures

A known failure mode of multi-persona prompting is that the personas converge to the same generic critique. Mitigations:

- **Disjoint rubrics, written into the prompt.** Each panelist's prompt section includes the explicit "out of scope for this panelist" line shown above.
- **Force-disagreement check.** After all four return verdicts, the panel-running subagent does a pass: "are any two panelists flagging the same issue under different names? If yes, deduplicate and assign to whichever panelist owns that section of the checklist."
- **Quoting requirement.** Each issue must quote a specific node, edge, or subgraph from the Mermaid source. Issues that can't be quoted are dropped — they're usually the generic ones.

## Scoring and stop condition

There is no aggregate score. Each panelist returns:

```json
{
  "panelist": "Trace Reader",
  "verdict": "revise",          // or "ship"
  "issues": [
    {
      "checklist_anchor": "missing-return-arrow",
      "quote": "user --> gw --> plan --> exec --> verify --> done[Done]",
      "explanation": "Trace ends at 'Done' without returning to user."
    }
  ]
}
```

The loop stops when any of:

1. **All-ship.** Every panelist returned `verdict: "ship"`.
2. **Budget.** `max_rounds` reached. Default 3 inline, 5 in the offline harness.
3. **Oscillation.** This round's issue set (by `checklist_anchor`) equals the previous round's. Means the generator can't fix what the panel is asking for; further rounds waste tokens.

On stop, return the latest draft plus the final panel report. If stopped by budget or oscillation, the report calls that out so the user knows the diagram isn't fully cleared.

## Where this plugs into the skill

### Stage 1 — hand-run via `references/panel-prompt.md`

A single markdown file at `references/panel-prompt.md` containing the full panel-critique prompt: the four panelist sections, the SME selection table, the anti-collapse instructions, and the JSON output contract. To use it, a developer:

1. Generates a diagram by running the skill normally (or grabs an existing one from `examples/`).
2. Opens a fresh Claude Code session, pastes the panel prompt, and pastes the diagram + plan beneath it.
3. Reads the JSON report Claude returns.
4. If anything was flagged, asks Claude to revise the diagram given the report, and optionally re-runs the panel.

That's the whole loop. No automation, no driver script. The point is to get five datapoints — one per eval in `evals/evals.json` — cheaply enough that we can decide whether the panel is doing real work before building any infrastructure around it.

If after the five hand-runs the panel is flagging issues that are real and that the single-shot skill missed, we proceed to stage 2. If it's flagging nothing, or flagging the same generic things every time, the loop is dead weight and we stop.

### Stage 2 — slash command (only if stage 1 earns it)

A markdown slash command under `.claude/commands/` (exact name TBD — `/refine`, `/critique-diagram`, etc.) that automates whatever stage 1 proved was useful. Likely shape:

- Reads the previous Claude response in the current conversation (or accepts a Mermaid block as an argument).
- Runs the panel critique as a subagent invocation, using the same `references/panel-prompt.md` we wrote in stage 1.
- Returns the JSON report inline; lets the user trigger a revise round explicitly rather than auto-revising.

Whether the same slash command should also have a mode that loops over `evals/evals.json` is a stage-2 question, not a now question.

The reason for the explicit gate between stages: a slash command is a small UX surface but still a surface — once it exists, changing it is friction. Stage 1 hand-runs cost almost nothing and tell us whether to commit.

## Subagent shape

A single subagent invocation that role-plays all four panelists, not four separate subagents. Reasons:

- **Cost.** One context window with the diagram and checklist loaded, four small persona sections, vs. four full setups.
- **Anti-collapse via cross-visibility.** With one subagent, we can include the explicit "are any two panelists saying the same thing under different names?" pass at the end. Four independent subagents can't do that without an aggregator step.
- **Karpathy's framing actually supports this.** "Channel/simulate many perspectives" is a *single-model* operation in his framing — the model doesn't have a "self" to begin with, so the question is what personas you ask it to simulate, not how many separate model calls you make.

The subagent's prompt structure:

```
You are running a four-panelist critique of a Mermaid system diagram.
Do not collapse the panelists into a single voice — each has a disjoint rubric.

[Diagram source pasted here]
[Plan section pasted here]
[Detected archetype: <archetype>]

PANELIST 1 — Trace Reader. Rubric: ... Out of scope for you: ...
PANELIST 2 — Visual Encoding Critic. Rubric: ... Out of scope for you: ...
PANELIST 3 — Scope Steward. Rubric: ... Out of scope for you: ...
PANELIST 4 — <SME for archetype>. Rubric: ...

For each panelist, return verdict + 0-3 issues with quoted source spans.
Then do the cross-visibility deduplication pass.
Return JSON.
```

## Tradeoffs and open questions

**Tradeoffs we accept:**

- Token cost. One inline round roughly doubles the per-request token count (generation + critique + revision). The offline harness multiplies more. Budget the default low (1 round inline) and let users opt into more.
- Latency. Adds a subagent round-trip; experience depends on whether the parent waits or streams.
- Persona collapse risk. Mitigated, not eliminated. Will need real measurement once the harness exists.

**Open questions, ordered by what we'd want to answer first:**

1. **Does the panel actually catch things the single-shot skill misses?** Cheapest test: hand-run `references/panel-prompt.md` against the five prompts in `evals/evals.json`, eyeball whether the panel flagged anything real. If the answer is "no, the first draft was already fine," the loop is dead weight and we stop before writing a slash command.
2. **Which archetype detector?** A small classifier prompt baked into the panel prompt? Keyword matching on the user's message? Or just have the panel prompt ask Claude to classify before assigning the SME? Probably the last — keeps it all in one prompt with no extra plumbing.
3. **Should stage-2 auto-revise, or just report?** Reporting is honest (the user sees what was flagged and decides), auto-revising is convenient but hides the panel's reasoning. Lean toward report-only first; auto-revise can be a flag later.
4. **What's the contract for "revise" when the user does ask for it?** Regenerate from scratch with the plan adjusted, or edit the prior Mermaid source in place? Editing in place preserves layout choices but risks layering revisions on top of structural mistakes. Lean toward full regeneration with the plan updated.

## Out of scope for this design

- Multi-level diagrams (system context → containers → components). The skill is single-level today and this loop doesn't change that.
- Non-Mermaid targets (D2, Structurizr, Graphviz). The panelists' rubrics are written against Mermaid idioms.
- Auto-tuning the rubrics from observed user feedback. That's a meta-loop we could build later, but we need a working first loop before we can tune it.
- Rendering / image-based critique. The panel reads Mermaid source, not pixels. A future panelist could be a vision-model "render reader" but that's a separate design.

## Stage 1 findings (2026-04-29)

**Status: stage 1 done. Stage 2 deferred.**

Stage 1 ran the hand-run plan above with one substitution — instead of running against all five `evals/evals.json` prompts, we ran against three real repos in `~/Github/` (an AI-agent project, a two-tower ML recommendation system, a Lambda-fronted GraphQL API). Plus a sanity-check run on a deliberately bad multi-tenant RAG diagram. Six total panel critiques across with-skill outputs.

### What worked

- **Disjoint rubrics held.** Trace Reader was silent on all six with-skill runs — empirical evidence that the skill's plan-first procedure consistently closes the loop and avoids dead-end nodes. The other three panelists each found 1–4 real issues per diagram while staying in their assigned lanes (no Trace Reader complaining about color, no Visual Encoding Critic complaining about return arrows).
- **Procedural counting on noun-inventory.** Forcing the panelist to count node declarations and cite the count caught 22-node sprawl on the AI-agent diagram (correctly hard-fail) and classified the 14-node ML diagram as soft-fail (correctly borderline). Without the explicit count step, an earlier rubric version vibe-checked an 18-node case as fine.
- **Generalized `choices-buried` anchor.** Initial wording targeted parenthetical text only; expanding it to cover `<br/>` subtitles and other in-label decoration caught real buried decisions on the GraphQL diagram (qs+body merge precedence, `data[root]` conditional shaping) and the ML diagram (L2 normalization, which is what makes FAISS IndexFlatIP equal cosine similarity).
- **The strongest catch class: `out-of-scope-sprawl`.** Twice across the runs, the panel caught the generator drawing things its own plan declared out of scope — once on the ML diagram (training pipeline drawn despite being fenced out), once on the GraphQL diagram (sibling-trace queryIds drawn despite being fenced out). This is the highest-value flag because the answer key was literally in the input.

### What didn't work

- **Cross-panelist dedup is unreliable on borderline cases.** Visual Encoding Critic flagging `decorative-color` and SME flagging `wrong-trust-surface` on the same out-of-axis subgraph survived dedup across both prompt iterations. On re-reading: this may actually be correct — color-doesn't-map-to-axis and subgraph-shouldn't-exist-on-axis are honestly different concerns even when they share a root cause architectural mistake. The worked example added to the prompt did not force a different outcome, and arguably shouldn't have. **Action:** treat dedup as best-effort, not deterministic. Don't claim it as a load-bearing feature.

### Known limitations to carry forward

1. **Single-run stochasticity is real.** The same diagram and same prompt produced different subsets of issues across two consecutive rounds — round 1 caught `out-of-scope-sprawl` on the ML diagram's offline pipeline, round 2 missed it. Single panel runs are not authoritative; coverage benefits from N runs unioned. Stage 2 should account for this if precision matters (run twice, union issue sets, or accept the noise as a feature of fast feedback).
2. **Borderline-anchor whack-a-mole.** Tightening one anchor (e.g. preventing `cross-cutting-as-peer` false-positives on properly-drawn sinks) sometimes pushes the same complaint to a different anchor (Scope Steward instead flagged the same dashboard as `out-of-scope-sprawl`, defensibly-but-pedantically). "Panel never flags anything on a clean diagram" is not the goal; the goal is that real issues outweigh noise, which six runs supports.
3. **Noun-inventory threshold is heuristic, not principled.** 12 / 13–16 soft / >16 hard is a reasonable default but the cutoffs aren't load-bearing. Genuinely complex systems may legitimately have >16 components on a single trace, in which case the right answer is "draw two diagrams at coarser granularity" — that lives in the skill's procedure, not in the panel rubric.

### Stage 2 status: deferred

Stage 2 (slash command) is not blocked, just unscheduled. The hand-runnable `references/panel-prompt.md` is sufficient for opportunistic use — paste it into any Claude Code session with a diagram, get a JSON report, decide whether to revise. A slash command earns its keep when this workflow becomes routine enough to deserve a stable UX surface; until then, the hand-run path is fine and lets us keep iterating on the prompt without locking in a command shape prematurely.

If/when stage 2 is taken up, the open questions in the section above (auto-revise vs report-only, regenerate vs edit-in-place, archetype detection) are still the right starting points. Add a fourth: should the slash command run the panel twice and union, given the stochasticity finding?
