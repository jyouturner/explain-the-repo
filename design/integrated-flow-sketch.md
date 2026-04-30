# Integrated flow sketch — panel as part of the default skill

Status: draft, pre-implementation. This sketches what `SKILL.md` would look like if the panel critique became a default step rather than a deferred opt-in slash command. Read alongside `design/refinement-loop.md` (the panel design itself).

The question this answers: instead of "user types `/refine` after a diagram," can we have "user types one thing and the skill produces a panel-refined diagram in one round-trip"?

## Proposed change to `SKILL.md`

Add a new step 6 to the procedure (currently stops at step 5). Update the output format (currently three things) to four things.

### New step 6 — Panel critique and one revision round

> After producing the diagram (steps 1–5), run a one-round panel critique before returning to the user. Spawn a subagent that reads `references/panel-prompt.md` and `references/diagnostic-checklist.md` and applies the four-panelist procedure to the diagram you just produced. Independent context matters — do not run the panel in your own context, since you'll grade work you just authored.
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

| Path                      | Calls | Notes                                              |
| ------------------------- | ----- | -------------------------------------------------- |
| Today (no panel)          | 1     | Generator only.                                    |
| Phase C, panel-clean      | 2     | Generator + panel critique (no revision needed).   |
| Phase C, revision-worthy  | 3     | Generator + panel critique + one revision.         |

So 2–3× tokens vs today, every invocation. If most diagrams come back panel-clean (which the stage-1 evidence weakly supports — Trace Reader was silent on all 6), the average is closer to 2×.

Latency adds the panel critique (which is small — single subagent, JSON output) plus optional revision. Estimated: +10–30 seconds per skill invocation depending on diagram size.

## What this commits us to

1. **Default-on means everyone pays the token cost,** including users who would have been happy with the single-shot output. Mitigations: panel-clean is the cheap path; users who don't want the panel can disable it in their skill settings (assuming we add a frontmatter flag — out of scope for this sketch).
2. **The filter table is now a load-bearing artifact.** Wrong calls in either direction hurt: a revision-worthy anchor in the borderline column lets bad diagrams through; a borderline anchor in the revision-worthy column triggers churn revisions on benign issues. Stage-1 evidence gives a starting point but the table will need tuning as more data accrues.
3. **Stochasticity becomes the user's problem.** A panel-clean response one minute and a flagged response the next on the same diagram will look like flakiness. We accept this and lean on the "panel summary" message to be honest about what was checked.
4. **The hand-runnable `panel-prompt.md` keeps working.** A user who wants a second opinion on an already-shipped diagram can still paste it in — phase C subsumes the manual path but doesn't break it.

## What this does NOT commit us to

- **Multi-round refinement.** Single round only. If users want more, they re-invoke the skill or hand-run the panel.
- **A slash command.** Phase C makes `/refine` largely redundant — the default flow already does what `/refine` would.
- **A new file format or storage of panel reports.** The summary is inline prose; the JSON is ephemeral and only surfaced on request.

## Open questions before we'd actually edit `SKILL.md`

1. **Should `cross-cutting-as-peer` be revision-worthy or borderline?** Stage 1 saw both real catches and a false positive. With the post-fix-1 rubric, false positives should drop, but we don't have N runs against the new rubric to confirm.
2. **Is the "spawn a subagent" requirement load-bearing?** Could the skill run the panel in its own context to save the overhead? Probably not — same-context grading was the bias I flagged early on, and we have no measurement that says it's safe.
3. **Should there be an env/setting flag to opt out?** A power user who knows what they want from a single shot shouldn't have to pay 2–3× tokens. But that's a feature flag and adds surface area.
4. **What does `iteration-mode-redraw` do under phase C?** The user shows an existing diagram and asks for a redraw. Does the panel run on the *new* diagram, the *original*, or both? Probably the new one only, but worth thinking about.

## Calibration findings (2026-04-29)

Ran phase C end-to-end against the three round-2 panel reports from stage 1: applied the (initial) filter table to partition issues, sent only the revision-worthy bucket to a reviser subagent, eyeballed the revised diagrams.

**Headline:** 2 strictly-better revisions (recs_two_tower, graphql-node), 1 regression (autoresearch). The regression was traceable to one specific cell of the filter table (`noun-inventory` hard-fail), which has been corrected above.

**recs_two_tower — clean win.** ARTIFACTS subgraph correctly folded into SERVING with explicit "loaded at construction" labels and offline-write arrows crossing the lifecycle boundary. The two-surface axis (offline / online) is now visibly the actual axis the diagram encodes. Borderline issues (FAISS/sklearn edge style, L2 normalize as buried choice, soft-band noun count) correctly left untouched.

**graphql-node — clean win.** `others` node and the dotted `sibling traces` edge removed; the same-shape sibling pattern moved to NOTES where out-of-scope acknowledgments belong. Borderline issues (sibling-traces edge style, three buried choices, gates-invisible) correctly left untouched.

**autoresearch — regression.** Two of three fixes were clean (`wrong-trust-surface` on DEEP, `out-of-scope-sprawl` on dashboard). The third (`noun-inventory` hard-fail at 22 nodes) caused the regression: the reviser collapsed `discover` + `triage` + `spec` + `single` + the `skills{}` decision diamond into a single `dd` node labeled with `·`-separated subtitles. **The new `dd` node is itself a `choices-buried` violation** — the orchestrated-vs-single-call dispatch (which the generator's plan named as a load-bearing architectural choice) is now hidden in a label. We fixed one anchor and created another.

**What changed in the filter table.** `noun-inventory` (both bands) reclassified from revision-worthy to borderline. Auto-consolidation under count pressure cannot reliably distinguish "filler nodes worth merging" from "structurally drawn architectural choices the plan named as load-bearing." Surfacing the count to the user — who has access to the plan's stated intent — is the safer call.

**Other observations from the calibration:**

- The borderline-vs-revision-worthy split worked as intended for everything except `noun-inventory`. Reviser correctly ignored borderline issues on all three cases.
- One-round-bound is a real risk: a second panel pass would have caught the autoresearch regression. Phase C ships this risk by design (more rounds → cost + stochasticity), and the panel summary surfaced to the user is the partial mitigation. Worth carrying as an open question.

## Suggested next step

With the filter table corrected, phase C is a defensible default-on change. Two ways to land it:

1. **Re-run the autoresearch reviser with the corrected filter** (only `wrong-trust-surface` and `out-of-scope-sprawl` as targets, no `noun-inventory`). Confirms the corrected table produces a strictly-better revision on the case that previously regressed. ~3 min. Strongly recommended before SKILL.md edit.
2. **Apply step 6 + the new output format to `SKILL.md`.** This is the actual phase C ship. Bound to one revision round; surface a panel summary; full JSON only on request. The hand-runnable `references/panel-prompt.md` keeps working unchanged.

After (2), the `design/refinement-loop.md` "Stage 2 deferred" note can be retired — phase C subsumes the stage-2 slash command.
