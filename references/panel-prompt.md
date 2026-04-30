# Panel critique prompt

A self-contained prompt for critiquing a Mermaid system diagram through four disjoint lenses, as designed in `design/refinement-loop.md`. Paste this file as context into a Claude Code session, paste the diagram (Mermaid source + plan + archetype) beneath it, and Claude returns a JSON panel report.

This is a stage-1 hand-run artifact. No automation is built around it yet. If the panel proves to catch real failures across the five evals in `evals/evals.json`, we proceed to stage 2 (a slash command). If not, we stop.

---

## How to use this file

1. Paste everything below the `--- BEGIN PROMPT ---` marker into a fresh Claude Code session.
2. Beneath the prompt, paste:
   - The Mermaid source block.
   - The plan (from step 2 of `SKILL.md`), or note "no plan provided".
   - The archetype hint, one of the six listed below, or `unknown`.
3. Read the JSON report Claude returns.
4. Optionally ask Claude to revise the diagram given the report, then paste the revised diagram back and re-run.

The five archetypes (from `evals/evals.json`):

- `request-trace-trust-bounds` — request trace where trust boundaries are the semantic axis (e.g. multi-tenant RAG)
- `request-trace-no-trust-bounds` — request trace where the axis is something else (e.g. CI/CD stages, automated vs human-gated)
- `cross-cadence-loop` — batch / nightly / training pipelines, not a request trace
- `iteration-mode-redraw` — diagnosing and redrawing an existing messy diagram
- `out-of-scope-non-system` — ERD, sequence-only, state machine, etc. (skill should have stepped aside)
- `unknown` — let the panel classify before critiquing

---

--- BEGIN PROMPT ---

You are running a four-panelist critique of a Mermaid system diagram. Each panelist owns a disjoint slice of the failure-mode taxonomy in `references/diagnostic-checklist.md` and `examples/anti-patterns.md`. Do not collapse the panelists into a single voice. A panelist whose slice is clean returns `verdict: ship` with an empty `issues` list — they do not borrow from another panelist's slice to look productive.

For every issue flagged:

- **Quote a specific span** from the Mermaid source — a node ID, an edge, or a subgraph header. Issues that cannot be quoted from the source are dropped before output.
- **Anchor to a checklist ID** from the lists below.
- **One sentence of explanation.** No more.
- **One anchor per underlying issue.** If two anchors from your own rubric describe the same problem at the same quoted span, pick the more specific anchor and drop the other. (This is a within-panelist dedup; the cross-panelist dedup pass is separate, below.)

After the four panelists return, run a cross-visibility deduplication pass:

- If two panelists flagged the same underlying issue under different names, keep it under whichever panelist owns that section of the checklist (per the anchor lists below) and remove it from the other.
- If a panelist's only issues all got removed in dedup, flip their verdict to `ship`.

If no plan was provided alongside the Mermaid source, that is itself a finding for the Trace Reader (anchor: `no-plan`).

If the archetype hint is `unknown`, classify it first by reading the prompt context the user provides; pick from the five named archetypes or fall back to "other" and pick the SME accordingly (see the SME table below).

---

## Panelist 1 — Trace Reader

**Lens:** Can a stranger follow this end to end without prior context?

**Rubric (anchors you may use):**

- `no-trace` — there is no clear entry point and exit point; the diagram shows components connected but no implied request, signal, or piece of data flowing through them.
- `missing-return-arrow` — a request goes in, no response comes out. The originator is shown initiating but never receiving anything back.
- `dead-end-nodes` — a node has incoming edges but no outgoing edges and is not a sink (audit log, metrics, storage). Or vice versa: a node emits but has no clear input.
- `multiple-cadences` — the diagram shows more than one time-scale at once (sync request flow + async batch retraining + failure-recovery loop). The reader cannot tell which arrows happen in milliseconds vs. minutes vs. days.
- `no-plan` — the user did not provide a plan; the trace cannot be verified against an intended entry/exit pair.

**Out of scope for this panelist:** color choices, layout, choice of subgraph axis, scope/inventory, domain accuracy. Flagging anything outside the rubric above is a panelist failure.

---

## Panelist 2 — Visual Encoding Critic

**Lens:** Does the visual encoding carry information, or is it decoration?

**Rubric (anchors you may use):**

- `decorative-color` — colors do not map to a single named semantic axis (trust boundary, control vs data plane, sync vs async, internal vs external, layer category). Two same-color nodes that don't share a category along that axis is a fail. If no semantic axis was named in the plan, default to fail.
- `layer-cake` — three or more horizontal bands stacked vertically with arrows mostly going down, suggesting the author drew abstraction levels rather than call structure.
- `unlabeled-cross-boundary` — an edge that leaves a subgraph and enters another (or leaves an internal region for an external one) without a label naming what flows on it.
- `inconsistent-edge-style` — solid edges (`-->`) and dotted edges (`-.->`) are mixed without a consistent convention. The expected convention: solid for the main trace, dotted for side effects (audit, lookup, async, observability).

**Out of scope for this panelist:** trace correctness, missing return arrows, scope/inventory, domain accuracy.

---

## Panelist 3 — Scope Steward

**Lens:** What's on the diagram that shouldn't be, and what's missing that the trace requires?

**Procedural step before applying this rubric:** count the node declarations in the source. A node declaration is any line introducing an identifier with a label — `id[label]`, `id{label}`, `id(label)`, `id([label])`, or `id[(label)]`. Cite the count in the explanation for `noun-inventory` if you flag it. Do not skip this count step.

**Rubric (anchors you may use):**

- `noun-inventory` — flag if any of:
  - node count exceeds **16** (hard ceiling — the diagram has slipped into completeness mode regardless of intent),
  - node count is **13–16** AND every component named or implied by the user's prompt appears as a box with roughly equal visual weight, no consolidation, no hierarchy of importance (soft fail — borderline-but-still-inventory),
  - regardless of count, every component named in the user's prompt appears as a box with equal weight (no selectivity at all).
  Cite the count and which of the three conditions fired. Counts of 12 or fewer do not fire this anchor on the count criterion alone.
- `cross-cutting-as-peer` — auth, PII redaction, audit logging, rate limiting, observability, metrics, or event streams drawn as a regular peer node in the main flow, with solid arrows that participate in call relationships. The following are CORRECT patterns and do NOT fire this anchor:
  - An edge label on the intercepted edge (e.g. `gw -->|"PII gate · model I/O"| llm`).
  - A gate glyph mid-edge.
  - A single sink node receiving multiple *dotted* incoming arrows from the writers.
  If the diagram uses any of these three patterns for the cross-cutter in question, the encoding is correct — do not flag.
- `choices-buried` — a real architectural decision (cost-aware tier selection, rules-then-LLM verification, PII redaction, lazy skill loading, manual-vs-automated gate, sync-vs-async dispatch) is hidden inside a node label — as parenthetical text, a `<br/>` second line, a subtitle suffix, or any other in-label decoration — rather than drawn structurally (split box, decision diamond, labeled fan-out, or label on the intercepted edge). The same architectural concern can fire this anchor on multiple boxes; flag each separately.
- `out-of-scope-sprawl` — the diagram is trying to also be the failure-path / batch-loop / observability diagram, instead of staying inside the one cadence and one scope the plan named.

**Out of scope for this panelist:** color, edge styling, layout, return arrows, domain accuracy.

---

## Panelist 4 — Domain SME (variable persona)

**Lens:** Does this look right for *this kind of system*?

Select the SME persona by archetype. Each SME has a small, domain-specific rubric. The SME may flag domain-specific issues not in the rubric, as long as they're domain-specific (not "this should be more colorful" — that's panelist 2's job).

| Archetype                          | SME persona                | Rubric                                                                                                                                                             |
| ---------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `request-trace-trust-bounds`       | Platform / SRE engineer    | `wrong-trust-surface`: subgraphs don't match real isolation surfaces. `tenant-shared-confused`: tenant-scoped vs shared-platform components placed in wrong group. |
| `request-trace-no-trust-bounds`    | Backend / API engineer     | `gates-invisible`: human-vs-automated or sync-vs-async gates not visible. `wrong-stage-order`: stages drawn in an order that doesn't match real call sequence.     |
| `cross-cadence-loop`               | ML / data engineer         | `cadence-unnamed`: the cadence (nightly, hourly, on-trigger) is not named in the plan or diagram. `serving-leak`: synchronous serving path mixed into a batch-pipeline diagram. `storage-not-cylindered`: warehouses, feature stores, registries drawn as boxes instead of cylinders. |
| `iteration-mode-redraw`            | Senior diagram reviewer    | `no-diagnosis`: redraw was produced without first naming 3-4 failure modes from the original. `regression`: the redraw reproduces one of the failure modes it was supposed to fix. |
| `out-of-scope-non-system`          | Mermaid generalist         | `wrong-sublanguage`: a flowchart was used where `erDiagram`, `sequenceDiagram`, or `stateDiagram-v2` is the correct Mermaid construct. `forced-trace`: a trace structure was imposed on something that is not a trace. |
| *fallback (archetype = other)*     | "Curious new hire"         | `cant-tell-what-it-does`: after reading, you cannot state in one sentence what this system does. `unanswered-obvious-question`: the most obvious question a reader would ask is not addressed by the diagram. |

The SME's rubric is intentionally smaller than the other three. If the SME finds nothing domain-specific to flag, return `verdict: ship` with empty issues — do not pad with generic critiques.

---

## Cross-visibility deduplication pass

After all four panelists have returned their verdicts and issues:

1. Group issues across panelists by the underlying problem (not by anchor — two anchors can name the same underlying issue).
2. For each group with more than one panelist's issue in it, keep the issue under the panelist whose section of the checklist owns that anchor (see the rubrics above) and remove it from the others.
3. For each panelist whose `issues` list is now empty, flip `verdict` from `revise` to `ship`.

Do not merge issues *within* a single panelist's list — those are theirs.

**Worked example.** The Visual Encoding Critic flags `decorative-color` on a subgraph because its color does not map to the named semantic axis. The Domain SME flags `wrong-trust-surface` on the same subgraph because it shouldn't exist as a peer group on that axis. Both issues quote the same subgraph and describe the same underlying problem — an out-of-axis subgraph was drawn. Resolution: keep it under the SME's `wrong-trust-surface` (more specific to subgraph structure than to color); remove the `decorative-color` issue. If `decorative-color` was the Visual Encoding Critic's only issue, their verdict flips to `ship`.

You must actually run this pass — it is not optional. If your output contains two issues from different panelists with quotes pointing at the same span and explanations describing the same root cause, you skipped the dedup.

---

## Output format

Return a single JSON object, no prose before or after. Schema:

```json
{
  "archetype": "request-trace-trust-bounds",
  "archetype_was_classified": false,
  "panel": [
    {
      "panelist": "Trace Reader",
      "verdict": "revise",
      "issues": [
        {
          "anchor": "missing-return-arrow",
          "quote": "user --> gw --> plan --> exec --> verify --> done[Done]",
          "explanation": "Trace ends at 'Done' instead of returning to user."
        }
      ]
    },
    {
      "panelist": "Visual Encoding Critic",
      "verdict": "revise",
      "issues": [
        {
          "anchor": "decorative-color",
          "quote": "classDef warm fill:#FAEEDA; class gw,llm,audit warm",
          "explanation": "gw, llm, and audit share a fill but don't share a category along the named trust-boundary axis."
        }
      ]
    },
    {
      "panelist": "Scope Steward",
      "verdict": "ship",
      "issues": []
    },
    {
      "panelist": "Domain SME (Platform / SRE engineer)",
      "verdict": "revise",
      "issues": [
        {
          "anchor": "tenant-shared-confused",
          "quote": "subgraph TENANT ... skills[Skill registry]",
          "explanation": "Skill registry is shared across tenants in the described system but is drawn inside the tenant-scoped subgraph."
        }
      ]
    }
  ],
  "stop_recommendation": "revise",
  "notes": "Three issues across three panelists; none are oscillating against a prior round (this is round 1)."
}
```

Field semantics:

- `archetype`: one of the six values listed earlier.
- `archetype_was_classified`: `true` if the input was `unknown` and the panel classified it; `false` if the user provided it.
- `panel`: exactly four entries, in the order Trace Reader → Visual Encoding Critic → Scope Steward → Domain SME. The SME entry's `panelist` field includes the persona in parentheses.
- `verdict`: `"ship"` or `"revise"`.
- `issues`: zero or more, each with `anchor` / `quote` / `explanation`.
- `stop_recommendation`:
  - `"ship"` if every panelist returned `ship`.
  - `"revise"` if any panelist returned `revise` AND this is round 1, OR the issue set differs from the previous round's.
  - `"stop-oscillation"` if the issue set (by `anchor`) is identical to the previous round's. (Only set this if the user told you a previous round's issue set; otherwise default to `revise`.)
  - `"stop-budget"` is set by the caller, not by you. Don't use it.
- `notes`: one or two sentences for the human reader. Keep terse.

If the diagram has no issues at all, all four panelists return `verdict: ship`, `issues: []`, and `stop_recommendation: ship`. That's the success case.

--- END PROMPT ---
