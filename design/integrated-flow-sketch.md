# Integrated flow sketch — panel as part of the default skill

Status: draft, pre-implementation. This sketches what `SKILL.md` would look like if the panel critique became a default step rather than a deferred opt-in slash command. Read alongside `design/refinement-loop.md` (the panel design itself).

The question this answers: instead of "user types `/refine` after a diagram," can we have "user types one thing and the skill produces a panel-refined diagram in one round-trip"?

## Proposed change to `SKILL.md`

Add a new step 6 to the procedure (currently stops at step 5). Update the output format (currently three things) to four things.

### New step 6 — Panel critique and one revision round

> After producing the diagram (steps 1–5), run a one-round panel critique before returning to the user. Spawn **two** subagents in parallel; each reads `references/panel-prompt.md` and `references/diagnostic-checklist.md` and applies the four-panelist procedure independently, in its own context. Independent context matters — do not run the panel in your own context, since you'll grade work you just authored. Union the flagged issues across both runs (collapse on identical anchor + quote; keep all distinct issues) before applying the filter.
>
> When the panel returns its JSON report, partition the issues into two buckets using the filter table below:
>
> - **Revision-worthy.** Revise the diagram once, then return.
> - **Borderline.** Do not revise; surface in the panel summary so the user can ask for a fix if they care.
>
> The revision is **bounded to one round**. Do not loop. Stochasticity (see `design/refinement-loop.md` § Known limitations) means a second round can introduce new issues without converging.
>
> If every panelist returned `verdict: ship`, skip the revision and surface a one-line "panel clean" note. If revision happens, surface the diagram, then a brief panel summary naming what was revised and what was left as borderline.

### Filter table — which anchors trigger revision

Based on stage-1 evidence (six panel runs, `design/refinement-loop.md` § Stage 1 findings):

| Anchor                       | Bucket          | Why                                                                                       |
| ---------------------------- | --------------- | ----------------------------------------------------------------------------------------- |
| `no-trace`                   | revision-worthy | Trace closure failure; high agreement across reviewers.                                   |
| `missing-return-arrow`       | revision-worthy | Same.                                                                                     |
| `dead-end-nodes`             | revision-worthy | Same.                                                                                     |
| `multiple-cadences`          | revision-worthy | Structural; the canonical anti-pattern this skill exists for.                             |
| `out-of-scope-sprawl`        | revision-worthy | Generator violating its own plan — strongest stage-1 catch class.                         |
| `wrong-trust-surface`        | revision-worthy | SME-flagged structural critique; substantive across runs.                                 |
| `tenant-shared-confused`     | revision-worthy | Same.                                                                                     |
| `cadence-unnamed`            | revision-worthy | Cadence labeling is cheap to fix and load-bearing for ML/batch diagrams.                  |
| `no-diagnosis`               | revision-worthy | Iteration-mode procedural failure.                                                        |
| `wrong-sublanguage`          | revision-worthy | Skill-stepped-aside failure; should redirect, not draw.                                   |
| `forced-trace`               | revision-worthy | Same.                                                                                     |
| `serving-leak`               | revision-worthy | ML-pipeline domain critique.                                                              |
| `regression`                 | revision-worthy | Iteration-mode redraw reproducing the original failure.                                   |
| `cross-cutting-as-peer`      | revision-worthy | Genuine peer-as-cross-cutter (with the post-fix-1 rubric, should now fire only when real).|
| `noun-inventory` (hard-fail) | borderline      | Cited count > 16. Was revision-worthy initially; calibration (2026-04-29) showed auto-consolidation introduces `choices-buried` violations on nodes the plan flagged as load-bearing architectural choices. Surface the count and let the user pick between consolidation or splitting into sibling diagrams. |
| `noun-inventory` (soft)      | borderline      | 13–16 band. Same reasoning as hard-fail; legitimate complexity is common in this band.    |
| `choices-buried`             | borderline      | Common; revising all surfaced choices structurally creates decision-tree spaghetti. Surface them and let the user pick which to redraw structurally. |
| `decorative-color`           | borderline      | Often a stretch when no axis was named (we saw round-1 do this on autoresearch).          |
| `unlabeled-cross-boundary`   | borderline      | Real catch sometimes, but cheap for the user to fix manually if they care.                |
| `inconsistent-edge-style`    | borderline      | Often pedantic — dotted/solid conventions are fuzzy by nature.                            |
| `gates-invisible`            | borderline      | Often a Mermaid limitation (no native fan-out/join glyph) more than a diagram defect.     |
| `wrong-stage-order`          | borderline      | Defensible simplification in many cases (saw this on graphql-node).                       |
| `storage-not-cylindered`     | borderline      | Surfaceable, low cost for the user to fix.                                                |
| `cant-tell-what-it-does`     | borderline      | Fallback SME persona's catch — surface, don't auto-act.                                   |
| `unanswered-obvious-question`| borderline      | Same.                                                                                     |
| `no-plan`                    | borderline      | Not a diagram defect; the user didn't supply a plan, often correctly (one-off).           |

A short rule of thumb backing the table: **revise on structural failures and clear plan-violations; surface on encoding/style/judgment calls and on count/threshold anchors.** The borderline bucket is the panel's "FYI" — useful, but not worth a token-spend revision. Count-based anchors (`noun-inventory`) belong in borderline because the user is in a better position than the reviser to judge whether a high count reflects real sprawl or legitimate complexity (see Calibration findings below).

### Updated output format

Currently SKILL.md says return three things. New version returns four:

> 1. **The plan** (from step 2).
> 2. **The Mermaid source** — post-revision, if revision happened.
> 3. **A brief note** about scope, surfaced choices, render config (unchanged).
> 4. **Panel summary** — at most 3 sentences. Examples:
>    - *"Panel clean — Trace Reader, Visual Encoding Critic, Scope Steward, and the SME (Backend / API engineer) all returned ship."*
>    - *"Panel flagged `out-of-scope-sprawl` (the offline pipeline was drawn despite being declared out of scope) — revised. Two borderline issues surfaced: `inconsistent-edge-style` on the FAISS/sklearn branch, and `noun-inventory` at 14 nodes (soft band)."*
>    - *"Panel revised one issue (`wrong-trust-surface` on the artifacts subgraph) and surfaced two borderline (`choices-buried` on L2 normalize; `noun-inventory` soft)."*
>
> The full JSON report is not surfaced by default; if the user wants it, they can ask. Surfacing every panel issue verbosely defeats the point of the filter.

## Cost: honest accounting

Per skill invocation:

| Path                      | Calls | Notes                                                              |
| ------------------------- | ----- | ------------------------------------------------------------------ |
| Today (no panel)          | 1     | Generator only.                                                                |
| Phase C, panel-clean      | 4     | Generator + 2 parallel panel critiques + 1 syntax linter.                      |
| Phase C, revision-worthy  | 5     | Generator + 2 parallel panel critiques + 1 syntax linter + one revision.       |

So 3–5× tokens vs today, every invocation. The two panel critiques and the syntax linter all run in parallel (see "Two-runs-unioned" and "Syntax-lint subagent" under Post-ship adjustments below). The linter is the cheapest of the four added calls — small JSON output, narrow scan; the dominant cost is the two panel critiques.

Latency adds the panel critique and the linter (both small JSON outputs; all three run in parallel so no sequential cost) plus optional revision. Estimated: +10–30 seconds per skill invocation depending on diagram size.

## What this commits us to

1. **Default-on means everyone pays the token cost,** including users who would have been happy with the single-shot output. Mitigations: panel-clean is the cheap path; users who don't want the panel can disable it in their skill settings (assuming we add a frontmatter flag — out of scope for this sketch).
2. **The filter table is now a load-bearing artifact.** Wrong calls in either direction hurt: a revision-worthy anchor in the borderline column lets bad diagrams through; a borderline anchor in the revision-worthy column triggers churn revisions on benign issues. Stage-1 evidence gives a starting point but the table will need tuning as more data accrues.
3. **Stochasticity becomes the user's problem.** A panel-clean response one minute and a flagged response the next on the same diagram will look like flakiness. We accept this and lean on the "panel summary" message to be honest about what was checked.
4. **The hand-runnable `panel-prompt.md` keeps working.** A user who wants a second opinion on an already-shipped diagram can still paste it in — phase C subsumes the manual path but doesn't break it.

## What this does NOT commit us to

- **Multi-round refinement.** Single round only. If users want more, they re-invoke the skill or hand-run the panel.
- **A slash command.** Phase C makes `/refine` largely redundant — the default flow already does what `/refine` would.
- **A new file format or storage of panel reports.** The summary is inline prose; the JSON is ephemeral and only surfaced on request.

## Open questions (resolved 2026-04-30)

All four open questions originally listed for phase C are now resolved. Captured here for posterity, with the resolution noted.

1. **Should `cross-cutting-as-peer` be revision-worthy or borderline?** Resolved: revision-worthy. Stage 1 fix-1 tightened the rubric so false positives on properly-drawn sinks (single sink + dotted incoming arrows) no longer fire. Six-run validation showed the post-fix rubric was net positive.
2. **Is the "spawn a subagent" requirement load-bearing?** Resolved: yes, keep the subagent. User confirmed during phase-C iteration. The marginal cost of a subagent invocation is small relative to the bias risk of self-grading; stays as the default mechanism.
3. **Should there be an env/setting flag to opt out?** Resolved: no flag. Instead, the skill respects user signals in the prompt (phrases like "quick diagram", "no panel", "fast", "skip the design phase") and drops to N=1 with no design-panel and no diagram-panel — but the syntax linter always runs since rendering correctness is non-negotiable. Documented in `SKILL.md` step 0 and step 6.
4. **What does `iteration-mode-redraw` do under phase C?** Resolved: step 5 now sends the redraw decision back to step 0, not step 1. The original sprawling diagram is often a single-diagram-overreach symptom; the design pass evaluates whether the redraw should be a multi-diagram set. Documented in `SKILL.md` step 5.

Two newer questions (introduced by post-ship adjustments) are also now resolved:

5. **Archetype tie-break when the user provides `unknown` and the two parallel runs classify differently.** Resolved: prefer the run whose Domain SME persona surfaced concrete domain-specific issues; if both SMEs were silent or flagged similar issues, prefer the first run. Documented in `SKILL.md` step 6.
6. **Revise = regenerate vs edit-in-place.** Resolved: regenerate from step 2's planning step with the panel issues injected as additional constraints in the plan's "Out of scope" list and step 3's mapping notes. Editing the prior Mermaid source in place is forbidden — calibration showed it layers revisions on top of structural mistakes. Documented in `SKILL.md` step 6.

The only remaining empirical question is **how the "is this complex enough to need siblings?" decision rule performs in real use**, which can only be answered through real-world traffic. That's not a question a design doc can answer.

## Calibration findings (2026-04-29)

Ran phase C end-to-end against the three round-2 panel reports from stage 1: applied the (initial) filter table to partition issues, sent only the revision-worthy bucket to a reviser subagent, eyeballed the revised diagrams.

**Headline:** 2 strictly-better revisions (recs_two_tower, graphql-node), 1 regression (autoresearch). The regression was traceable to one specific cell of the filter table (`noun-inventory` hard-fail), which has been corrected above.

**recs_two_tower — clean win.** ARTIFACTS subgraph correctly folded into SERVING with explicit "loaded at construction" labels and offline-write arrows crossing the lifecycle boundary. The two-surface axis (offline / online) is now visibly the actual axis the diagram encodes. Borderline issues (FAISS/sklearn edge style, L2 normalize as buried choice, soft-band noun count) correctly left untouched.

**graphql-node — clean win.** `others` node and the dotted `sibling traces` edge removed; the same-shape sibling pattern moved to NOTES where out-of-scope acknowledgments belong. Borderline issues (sibling-traces edge style, three buried choices, gates-invisible) correctly left untouched.

**autoresearch — regression.** Two of three fixes were clean (`wrong-trust-surface` on DEEP, `out-of-scope-sprawl` on dashboard). The third (`noun-inventory` hard-fail at 22 nodes) caused the regression: the reviser collapsed `discover` + `triage` + `spec` + `single` + the `skills{}` decision diamond into a single `dd` node labeled with `·`-separated subtitles. **The new `dd` node is itself a `choices-buried` violation** — the orchestrated-vs-single-call dispatch (which the generator's plan named as a load-bearing architectural choice) is now hidden in a label. We fixed one anchor and created another.

**What changed in the filter table.** `noun-inventory` (both bands) reclassified from revision-worthy to borderline. Auto-consolidation under count pressure cannot reliably distinguish "filler nodes worth merging" from "structurally drawn architectural choices the plan named as load-bearing." Surfacing the count to the user — who has access to the plan's stated intent — is the safer call.

**Other observations from the calibration:**

- The borderline-vs-revision-worthy split worked as intended for everything except `noun-inventory`. Reviser correctly ignored borderline issues on all three cases.
- One-round-bound was a real risk: a second panel pass would have caught the autoresearch regression. Initially shipped as a known limitation; subsequently addressed by adopting two-runs-unioned (see Post-ship adjustments below). The single-revision bound on revision *itself* is unchanged.

## Post-ship adjustments

### Two-runs-unioned (adopted 2026-04-29)

Step 6 now spawns two panel-critique subagents in parallel rather than one, and unions their flagged issues (by `(anchor, quoted span)`) before applying the filter table.

**Motivation.** Stage-1 evidence and the calibration run both showed single panel runs are not authoritative — consecutive runs against the same diagram surface different subsets of issues (canonical case: the recs_two_tower offline-pipeline `out-of-scope-sprawl` flag, caught in round 1 but missed in round 2). A second pass would also have caught the autoresearch reviser regression that motivated the calibration's filter-table fix in the first place.

**Cost.** Two parallel panel critiques double the panel-step tokens but add no wall-time (they run concurrently). Total cost moves from 2–3× → 3–4× the no-panel baseline; see updated cost table above.

**Trade-off accepted.** Doubling token spend for stochasticity coverage is a real cost. The argument for default-on rather than opt-in: panel-clean is rare in practice (stage-1 saw only Trace Reader returning ship cleanly across all six runs; the other panelists found something almost every time), so the marginal cost of a second run is paid mostly when it actually adds coverage rather than just confirming a clean result.

**Smaller question this introduces.** When the user provides `unknown` as the archetype hint and the two parallel runs classify it differently, which classification wins? Current handling: prefer whichever picked an SME persona that surfaced domain-specific issues; otherwise first-run. Not yet validated against real divergent-classification cases.

### Syntax-lint subagent (adopted 2026-04-29)

Step 6 now spawns a third subagent in parallel with the two panel critiques: a syntax linter that reads `references/syntax-lint-prompt.md` and scans the Mermaid source for parser-blocking footguns (unquoted special chars in edge labels, `@` in non-basic node shapes, subgraph display names without quotes, reserved keywords as IDs, the markdown-list trap, linkStyle index out-of-range).

**Motivation.** Recurring failure mode in AI-generated Mermaid: a single unquoted `[]` or `@` causes GitHub's renderer to fail with "Unable to render rich display" while Mermaid Live surfaces a parse error pointing at the offending line. The phase-C panel critique focuses on semantic and structural issues; it doesn't run the source through a parser. The `examples/real-repos/graphql-node.md` diagram shipped with `norm -.->|products[]| resolver` and broke on GitHub — the patterns reference (`references/mermaid-patterns.md` § Syntax footguns) had a relevant section, but as a passive reference it didn't enforce. A procedural linter does.

**Mechanism.** The linter returns `auto_fixes` as `(find, replace)` pairs the skill applies verbatim before partitioning panel issues. `manual_fixes` (e.g., reserved-keyword node IDs that would require renaming all references) are surfaced to the user. Verified end-to-end against the `products[]` case: linter correctly flagged the only parser-blocking issue and produced no false positives on stylistic-but-parseable patterns elsewhere in the same source.

**Cost.** Adds one small subagent call per invocation. Total cost moves from 3–4× → 3–5× the no-panel baseline; the linter is the cheapest of the four added calls (small JSON, narrow scan).

**What this does NOT replace.** The patterns reference still has authority for diagram-authoring guidance during generation — the linter is the post-hoc safety net, not the primary teacher.

### Multi-diagram sets and the design phase (adopted 2026-04-29)

Phase C originally scoped the skill to "one diagram, well drawn." User feedback after the first calibration runs surfaced a real limitation: complex systems (multi-cadence agents, multi-loop pipelines, components with load-bearing internal structure) sprawl when forced into a single trace, and the panel correctly flags the sprawl but the reviser can't actually fix it without losing structural detail. The autoresearch case ended at 21 nodes because that's what the system honestly needs in one frame — collapsing it bury what matters.

The complete fix is to generate a **diagram set** when the system warrants one, not a single diagram. Step 0 of `SKILL.md` is the design pass that decides set composition; `references/design-pass.md` contains the sibling rules; `references/design-panel-prompt.md` is the set-level critique that runs when N > 1.

**Sibling rules** (full version in `references/design-pass.md`):

- **Cadence sibling:** sync vs async vs scheduled flows that don't share a time scale.
- **Failure sibling:** failure / recovery path structurally distinct from happy path.
- **Zoom sibling:** a load-bearing component on the headline trace gets its own diagram exploding it.
- **Topology sibling:** state stores / deployment topology that the trace doesn't naturally show.
- **Lifecycle sibling:** distinct build-time and run-time concerns (training + serving).

Default set size is 1. Cap at 5. Most systems land at 1–3; agent / multi-cadence systems land at 4–5.

**Design-panel rubric** covers composition-level failures:

- `missing-aspect` (revision-worthy)
- `wrong-cadence-split` (revision-worthy)
- `unanchored-zoom` (revision-worthy)
- `redundant-siblings` (borderline; surface)
- `over-fanout` / `under-fanout` (borderline; surface)
- `headline-unclear` (borderline; surface)

The design-panel runs once per request on the *set design*, before any per-diagram generation. Skipped entirely for single-diagram sets.

**Cost.** Single-diagram sets (most common case for simple systems) pay the same 3–5× as before. Multi-diagram sets pay roughly N× that. Updated cost table:

| Path                                            | Calls       | Notes                                                                  |
| ----------------------------------------------- | ----------- | ---------------------------------------------------------------------- |
| Today (no panel)                                 | 1           | Generator only.                                                        |
| Phase C, single-diagram set, panel-clean         | 4           | Generator + 2 panel + 1 linter.                                        |
| Phase C, single-diagram set, panel revises       | 5           | Above + one revision.                                                  |
| Phase C, N-diagram set (N>1), all clean          | 1 + N × 4   | 1 design-panel + N × (generator + 2 panel + 1 linter).                 |
| Phase C, N-diagram set (N>1), all revise         | 1 + N × 5   | Above + N × revision.                                                  |

For N=4 worst case: 21 calls. Wall-time doesn't scale at the same rate (per-diagram subagents run in parallel within each diagram; diagrams themselves can be generated in parallel if independent), but tokens do. The trade-off accepted: a coherent multi-diagram set is worth more than a single sprawling diagram, even at this cost. The user gets one chance to dismiss the design pass at step 0 ("just give me one diagram") if they want the cheap path.

**What's NOT in this expansion.** The skill is still single-level (C4 "container" zoom, roughly). The design phase doesn't address true multi-level diagramming (system context → containers → components → code). Tools like Structurizr / C4-PlantUML are still the right answer for that.

**Remaining limitations to validate against real use:**

1. The "is this complex enough to need siblings?" decision rule is heuristic. The first wave of multi-diagram outputs in real use will show whether step 0 over-fans-out or under-fans-out for typical user requests.
2. Multi-diagram output is verbose. The `## Diagram N — <title>` section pattern may need tuning if users find it too long.
3. The `examples/real-repos/autoresearch.md` is currently single-diagram; it's a candidate for regeneration as a multi-diagram example to demonstrate what the new flow produces. Not blocking — the framework ships independently.

## Suggested next step

With the filter table corrected, phase C is a defensible default-on change. Two ways to land it:

1. **Re-run the autoresearch reviser with the corrected filter** (only `wrong-trust-surface` and `out-of-scope-sprawl` as targets, no `noun-inventory`). Confirms the corrected table produces a strictly-better revision on the case that previously regressed. ~3 min. Strongly recommended before SKILL.md edit.
2. **Apply step 6 + the new output format to `SKILL.md`.** This is the actual phase C ship. Bound to one revision round; surface a panel summary; full JSON only on request. The hand-runnable `references/panel-prompt.md` keeps working unchanged.

After (2), the `design/refinement-loop.md` "Stage 2 deferred" note can be retired — phase C subsumes the stage-2 slash command.
