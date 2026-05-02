---
name: explain_the_repo
description: Produce architecture documentation that explains a codebase — what the system does, how it's built, how data flows, what persists, where to start reading. Combines prose sections (component summaries, data flow descriptions, deployment notes, glossary, runbook entries) with multi-diagram Mermaid sets generated through a doc-level design pass and per-section critique. Use this skill whenever the user asks for an architecture doc, system explanation, "what does this codebase do," repo overview, onboarding doc, or "help me understand [project]." For diagrams alone (e.g. "draw a diagram of X"), the inner diagram procedure runs standalone.
---

# Explain the repo

A skill for producing architecture documentation that explains a codebase to a stranger. The output is a single markdown file that combines prose sections with Mermaid diagram sets where visual representation is load-bearing.

## Why this skill exists

The default failure mode for architecture documentation is one of two extremes: either a wall of prose with no visual grounding, or a single sprawling diagram with no surrounding explanation. Both fail the stranger trying to onboard. A good architecture doc interleaves prose (component responsibilities, design intent, where to start reading) with diagrams (request flow, state landscape, lifecycle boundaries) — each load-bearing in its own way, each grounded in specific files in the codebase.

This skill encodes a procedure that designs the doc as a whole before writing any of it: which sections are needed, which are prose vs diagrams, what each grounds in, what's deliberately out of scope. The diagrams themselves use a focused inner loop (`references/diagram-procedure.md`) that produces multi-diagram sets with their own critique pass. A locked visual vocabulary holds across every diagram in the doc so a reader doesn't relearn a color/shape mapping in every section. A reader-test phase (Phase D) runs a fresh subagent against the produced doc to confirm a stranger can actually answer onboarding questions from it. The result is a coherent doc, not a collection of artifacts — and one whose ability to onboard is measured, not assumed.

## When to use this skill

Trigger on requests like:

- "Explain this repo / codebase / project"
- "Architecture of [project]" / "system overview of [service]"
- "What does this codebase do" / "how does this work end to end"
- "Onboard a new engineer to [repo]" / "documentation for the team"
- "Document this service" / "give me an architecture doc"
- "Help me understand [project]"

For requests that are *only* about a diagram ("draw a diagram of X", "request flow for Y"), the inner diagram procedure (`references/diagram-procedure.md`) can be invoked directly without the doc-level wrapper. The same procedure produces visual sections inside the doc and standalone diagrams.

Do NOT use this skill for:

- API references — use OpenAPI / typedoc / language-specific tools.
- Code-level documentation (function-by-function, class-by-class) — different shape, different audience.
- README files for end users (install, usage, examples) — same file format, different content.
- Non-software repos — the section taxonomy assumes a system being explained.

## The procedure

Five phases plus a pre-flight. **Phase 0** scans the environment (existing docs in the repo, whether the current session authored the system being documented). **Phase A** is the doc design pass — decide what sections the doc needs and lock the visual vocabulary. **Phase B** is per-section generation — for prose sections, write prose; for diagram sections, follow the inner diagram procedure. **Phase C** is doc-level review. **Phase D** is the reader test — a fresh subagent reads the doc and answers five onboarding questions to confirm the doc actually onboards. **Phase E** is the output.

### Phase 0 — Pre-flight

Before the doc design pass, two scans inform the rest of the procedure.

#### Phase 0a — In-session-author detection

Check whether the current Claude session has authored the system being documented. Heuristics, in order:

- **Session-edited source files.** Have files in the target repo's source tree been edited within this conversation? If yes, weight strongly toward "in-session author."
- **Conversation length on this codebase.** A long task chain in this session that produced the codebase indicates the agent is the author.
- **Memory entries.** Persistent memory entries describing recent decisions on this repo indicate the agent has substantial prior context.
- **Recent git activity.** `git log --since='2 hours ago' --author=Claude` returns commits — strong "in-session" signal.

If detected:

1. **Skip the panel critique loops** — Phase A's optional doc-panel, Phase B's per-diagram panel critiques, Phase C's doc-level panel. Self-grading in the same context is unreliable even with subagents because the parent session's recent context can leak into spawned subagents through summarization, system prompt, and environment.

2. **Do NOT skip the syntax linter or the reader test.** The syntax linter is mechanical and unaffected by context bias. The reader test runs in a fresh subagent with no source-code access, which is exactly the protection the test is designed to provide — its independence is intrinsic, not contingent.

3. **Emit the in-session disclosure block** in Phase E's generation notes (see Phase E template). Don't leave this to the agent's discretion — emit it whenever Phase 0a triggers.

If the heuristic is uncertain (some signal but not strong), ask the user explicitly:

> "Did this conversation author significant parts of this repo? If yes, I'll skip the panel critiques to avoid self-grading. The reader test still runs."

If the user says no, run the full procedure including panel critiques.

#### Phase 0b — Existing-docs scan

Scan the target repo for existing documentation:

- `README.md` at repo root
- `ARCHITECTURE.md` at repo root
- `CONTRIBUTING.md` at repo root
- `docs/` folder contents (every `*.md` file, recursively)
- `*.md` files in source subdirectories where the file is more than ~50 lines (skip stub READMEs)

For each found doc, summarize in one sentence what it covers. Pass the summaries into Phase A as the "Existing docs detected" input — the doc plan will use them to position the new doc complementarily, not redundantly. See `references/doc-design-pass.md` § "Format for the doc plan".

If any existing doc would substantially overlap the proposed new doc (>50% same content area), surface a question to the user before generation:

> "Existing `ARCHITECTURE.md` already covers schemas and extension points. Should I (a) position the new doc as a stranger-onboarding companion (where-to-start, lifecycle, glossary), (b) replace the existing one, or (c) merge into the existing one?"

Don't unilaterally produce a duplicate. The default is (a) — complementary positioning — unless the user picks otherwise.

### Phase A — Doc design pass

Read the repo (or accept the user's description). Decide what sections the doc needs and lock the visual vocabulary.

Read `references/doc-design-pass.md` in full — it has the section taxonomy (architecture overview, data flow, state and persistence, lifecycle, failure modes, test-and-eval-surface, glossary, where-to-start, etc.), the decision rules (which sections this kind of repo needs), the merge rule for lifecycle/state-and-persistence, the required template for component summaries, the visual-vocabulary lock-in procedure, and the format for the doc plan.

Default doc shape: headline + architecture overview + where-to-start. Add sections per the decision rules. Cap at 8–10 sections — beyond that, the doc becomes a documentation site rather than a single explainer.

Produce the doc plan as a short prose block, including the "Existing docs detected" item (from Phase 0b) and the locked visual vocabulary. Show it to the user and ask for a quick sanity check before generation.

If the doc plan has more than 3 sections AND Phase 0a did not trigger in-session-author detection, run the doc-panel critique on the plan: spawn a subagent that reads `references/doc-panel-prompt.md` and returns a JSON report on the doc composition. Apply revision-worthy issues by adjusting the plan before generation; surface the rest in the doc-level summary.

If the doc has 3 or fewer sections, skip the doc-panel.

If Phase 0a detected in-session authorship, skip the doc-panel regardless of section count. The reader test (Phase D) still runs.

#### Phase A's token-budget output

After the doc plan is finalized but before generation begins, output an estimated token cost so the user can opt for the lighter path informed:

```
Doc plan finalized. Estimated token cost (assuming N sections, M total diagrams):

  Phase A doc-panel critique:           ~2K tokens (skipped if ≤3 sections OR in-session)
  Phase B per-section generation:       ~6K tokens × N sections          = ~<6K·N>
  Phase B per-diagram panel (×2 critics): ~6K tokens × M diagrams        = ~<6K·M>  (skipped if in-session)
  Phase B per-diagram syntax linter:    ~500 tokens × M diagrams         = ~<500·M>
  Phase C doc-level panel:              ~3K tokens (skipped if ≤3 sections OR in-session)
  Phase D reader test:                  ~3K tokens
  Phase E output assembly:              ~1K tokens

  Full path total:                      ~<sum> K tokens
  Light path (linter + reader test only): ~<sum-light> K tokens
```

Numbers are rough; report ranges when uncertain. The point of this output is decision support, not accuracy. Show it to the user and offer the choice:

> "Proceed with full path (~XX K tokens), light path (~YY K tokens, skips Phase B/C panels), or cancel?"

If the user picks the light path, skip the per-diagram panel critiques and the doc-level panel, but keep the syntax linter and the reader test. The reader test is the minimum non-skippable critique loop because it's the only one that simulates the stranger-onboarding experience the doc exists to serve.

**Opt-out.** If the user signals they want a quick doc — phrases like "rough overview", "skip the design pass", "just sketch it", "no critique" — drop to a minimal three-section doc (headline + one diagram set + where-to-start), skip the doc-panel and per-diagram panels, but **still run the reader test and the syntax linter** unless the user has explicitly opted out of the reader test too. Rendering correctness and stranger-onboarding fitness are both non-negotiable.

### Phase B — Per-section generation

For each section in the plan, generate it. Section types:

- **Prose section.** Write the prose. Ground every claim in specific files / functions in the codebase. Cite paths (`src/foo.py:42`, `lib/bar.ts`). Don't fabricate. If the source isn't clear from your reading, surface it as a `TODO: confirm in <file>` rather than inventing a plausible-sounding answer. The prose's job is to explain *intent and structure*, not to repeat what the code already says.

- **Diagram-set section.** Follow the procedure in `references/diagram-procedure.md`. The diagram procedure has its own design pass (which diagrams in the set, how they relate), per-diagram generation, panel critique, syntax linter, and bounded revision. The output is the diagram set with its panel summaries, ready to embed in the doc section.

- **Hybrid section.** Prose + diagram set. The prose introduces or follows the diagram(s); both ground in the same source files. Write the prose first (so the diagrams have something to anchor to), then generate the diagrams.

Sections can be regenerated independently if the doc-panel flags one as broken without affecting the others.

### Phase C — Doc-level review

After all sections are generated and assembled, spawn a subagent that reads `references/doc-panel-prompt.md` and reviews the assembled doc. The doc-level panel checks coherence — do sections agree? do pointers resolve? is anything missing across the whole? Is the visual vocabulary consistent across diagrams? Is the component-summary template applied? Did the eval surface get a section if it earned one? Does the lifecycle/state overlap need merging? It does NOT re-check what the diagram-level panel already covered (that's the inner loop's job).

If the doc-panel returns revision-worthy issues, revise once. Bounded — do not loop. The revision regenerates only the affected sections, not the whole doc.

Skip Phase C if Phase A's doc-panel was skipped (3-or-fewer-section doc, opt-out, or in-session author detected). Phase D's reader test still runs.

### Phase D — Reader test

After the panel critiques (Phase C) have run and any revisions are applied — or whenever Phase C was skipped — run the reader-test pass. This is the eval-driven analog of the panel critiques: the panels check that the doc has the right shape; the reader test checks that a stranger can actually use the doc to answer onboarding questions. Both matter; neither replaces the other.

**This phase runs unconditionally** unless the user has explicitly opted out of the reader test specifically. It runs even when in-session authorship was detected (the reader test's protection is intrinsic — the subagent has no source-code access — so the in-context concern doesn't apply). It runs even on 3-section docs (short docs are exactly the ones at risk of "looks fine, doesn't onboard").

**Procedure.**

1. Generate the five reader-test questions for this specific doc:
   - Q1 (universal): "If I'm new to this codebase, what file should I read first and why?"
   - Q2 (per-doc): "What does `<X>` do, and what does it deliberately *not* do?" — pick `<X>` as a load-bearing component named in the doc (typically the one with the most edges in the architecture-overview diagram, or the first component in the component-summaries section).
   - Q3 (per-doc): "How does data move from `<input>` to `<output>` through the system?" — pick `<input>` and `<output>` as the boundaries of the headline trace.
   - Q4 (universal, with substitution): "What persists between runs of this system, and where does it live?" If the doc deliberately omitted state-and-persistence, substitute: "What's the system's relationship to persistent state, and is there any?"
   - Q5 (universal): "What is deliberately out of scope for this system, and where would I look for that information instead?"

2. Spawn a fresh subagent **with no access to the source repo and no prior conversation context** — this is load-bearing for the test's validity. Hand it `references/reader-test-prompt.md` along with the produced doc and the five questions.

3. Read the JSON report the subagent returns.

4. Apply the verdict per the filter in `references/reader-test-prompt.md`:

   - `pass`: ship. Surface "reader test: 5/5 answered" in generation notes.
   - `borderline`: ship. Surface the partial answers and their gaps in generation notes; do not revise.
   - `revise`: regenerate the section(s) corresponding to the failed question(s). Bounded to one round — do not re-run the reader test after revision. Surface the original gaps and what was changed in generation notes.

The reader test is bounded to one round. Re-running it after revision risks turning the test into a tuning loop, which defeats its purpose as an independent check. If the bounded revision didn't fix everything, surface the remaining gaps in generation notes and ship.

### Phase E — Output

Return the complete markdown file as one artifact. The user should be able to save it directly as `docs/architecture.md` (or wherever the repo's docs live) without further editing.

The generation notes go at the very end of the doc, in a `<details>` block titled "Generation notes" so they don't clutter the doc itself but stay auditable. The notes follow this template:

```
<details>
<summary>Generation notes</summary>

**Doc plan.** <N sections, names>. Existing docs detected: <list, or "none">.
Visual vocabulary: <colors and shapes locked at design pass>.

**Phase 0a — In-session-author detection.** <triggered: panel critiques skipped | not triggered: full procedure>.
<If triggered:> The doc was produced by the same Claude session that authored the system. Panel critiques (doc-level and per-diagram) were skipped because in-context self-grading is unreliable; the syntax linter and reader test still ran. To get the full panel-critique procedure, invoke this skill in a fresh session with no prior context on this repo.

**Phase A doc-panel critique.** <ship | revised one issue (anchor: explanation) | skipped (3-or-fewer-section doc | in-session | opt-out)>.

**Phase B per-section diagram panels.** <inline summary if any sections had diagrams; per-diagram ship/revised note>. <Or: skipped (in-session | opt-out).>

**Phase B syntax linter.** <all clear | N auto-fixes applied | M manual fixes surfaced (categories)>.

**Phase C doc-level panel.** <ship | revised one issue (anchor: explanation) | skipped (in-session | opt-out)>.

**Phase D reader test.** <pass: 5/5 answered | borderline: K/5 answered, partial gaps in qN | revised qN section after cant-answer | skipped per user opt-out>.

</details>
```

The "Phase 0a" disclosure is **required** when in-session authorship was detected. Don't omit it — the disclosure is the user's signal that the doc was produced under conditions that limit panel-critique validity. Future readers (including the user when they come back to this doc later) deserve to know.

## Output format

A single markdown file with this skeleton:

```
# <System name> — architecture

<Headline paragraph: what this is, in 2–3 sentences. Names the system, its
purpose, and the most architecturally interesting property.>

## Where to start reading

<Pointers into the codebase: entry points, key interfaces, the file that
explains the most. 3–6 bullets, each with a path and one sentence.>

## Architecture overview

<Diagram set or single diagram + prose. The headline explanation of how
the system works at the system level.>

## <Other sections per the doc plan>

...

## Out of scope

<What this doc deliberately doesn't cover, and where to find that
information instead. One or two sentences each.>

<details>
<summary>Generation notes</summary>

Doc plan: <N sections, names>.
Doc-panel: <ship | revised one issue (...) | skipped (3-or-fewer-section doc)>.
Per-section diagram panels: <inline summary if any sections had diagrams>.
Syntax linter: all clear.

</details>
```

The "Where to start reading" section is non-optional. Every doc gets one. It's the single highest-leverage section for a stranger.

## Common failure modes to avoid

- **Fabricating specifics.** Don't write component descriptions that aren't grounded in the code. If something's unclear from your read of the repo, mark it as `TODO: confirm in <file>` rather than producing plausible-sounding fiction.
- **Repeating the README.** The architecture doc has a different shape — focus on call structure, data flow, persistence, and "where to read what next," not on install instructions or feature lists. If the project has a README, the architecture doc complements it, doesn't replace it.
- **Drafting Mermaid inline without the procedure.** When a section needs a diagram, follow `references/diagram-procedure.md`. The inner procedure exists because freeform Mermaid drafting has predictable failure modes.
- **Producing a doc that's just diagrams + captions.** The prose sections are load-bearing. A reader of an architecture doc needs both the structural shape (diagrams) and the design intent (prose).
- **Skipping the where-to-start section.** A stranger reading the doc has nowhere to land in the code without it.
- **One sprawling 14-section doc.** If the doc plan has more than 8–10 sections, that's a documentation site, not a single doc — surface the question to the user (do you want a doc set, or a leaner single doc?).

## Reference files

- `references/diagram-procedure.md` — the inner loop for generating diagram sets (used by Phase B for diagram-set sections; usable standalone). Self-contained: includes its own design pass, per-diagram generation, panel critique, syntax linter, and bounded revision. Now also receives the locked visual vocabulary from Phase A.
- `references/doc-design-pass.md` — section taxonomy and decision rules for Phase A (which sections does the doc need). Includes the component-summary required template, the visual-vocabulary lock-in procedure, the lifecycle/state-and-persistence merge rule, and the test-and-eval-surface section type.
- `references/doc-panel-prompt.md` — doc-level critique used by Phase A (when N > 3 and not in-session) and Phase C. Anchors include `inconsistent-vocabulary`, `component-summary-template-violation`, `eval-surface-buried`, and `lifecycle-state-overlap-not-merged`.
- `references/reader-test-prompt.md` — the Phase D reader-test prompt. A fresh subagent reads the doc cold (no source-code access) and answers five archetypal onboarding questions; the test's verdict drives a bounded one-round revision.
- `references/diagram-set-design-pass.md` — within-section rules for deciding the diagram-set composition (used by the inner diagram procedure).
- `references/diagram-set-panel-prompt.md` — within-section diagram-set critique (used by the inner diagram procedure).
- `references/panel-prompt.md` — per-diagram four-panelist critique (used by the inner diagram procedure).
- `references/syntax-lint-prompt.md` — Mermaid parser-footgun checklist (used by the inner diagram procedure).
- `references/diagnostic-checklist.md` — diagram-level failure modes for iteration mode.
- `references/mermaid-patterns.md` — Mermaid idioms and tactical guidance. Includes the "Publishing to GitHub README" section with dagre-survival constraints.
