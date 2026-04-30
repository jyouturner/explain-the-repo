---
name: explain_the_repo
description: Produce architecture documentation that explains a codebase — what the system does, how it's built, how data flows, what persists, where to start reading. Combines prose sections (component summaries, data flow descriptions, deployment notes, glossary, runbook entries) with multi-diagram Mermaid sets generated through a doc-level design pass and per-section critique. Use this skill whenever the user asks for an architecture doc, system explanation, "what does this codebase do," repo overview, onboarding doc, or "help me understand [project]." For diagrams alone (e.g. "draw a diagram of X"), the inner diagram procedure runs standalone.
---

# Explain the repo

A skill for producing architecture documentation that explains a codebase to a stranger. The output is a single markdown file that combines prose sections with Mermaid diagram sets where visual representation is load-bearing.

## Why this skill exists

The default failure mode for architecture documentation is one of two extremes: either a wall of prose with no visual grounding, or a single sprawling diagram with no surrounding explanation. Both fail the stranger trying to onboard. A good architecture doc interleaves prose (component responsibilities, design intent, where to start reading) with diagrams (request flow, state landscape, lifecycle boundaries) — each load-bearing in its own way, each grounded in specific files in the codebase.

This skill encodes a procedure that designs the doc as a whole before writing any of it: which sections are needed, which are prose vs diagrams, what each grounds in, what's deliberately out of scope. The diagrams themselves use a focused inner loop (`references/diagram-procedure.md`) that produces multi-diagram sets with their own critique pass. The result is a coherent doc, not a collection of artifacts.

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

Two phases. **Phase A** is the doc design pass — decide what sections the doc needs. **Phase B** is per-section generation — for prose sections, write prose; for diagram sections, follow the inner diagram procedure. **Phase C** is doc-level review.

### Phase A — Doc design pass (step 0)

Read the repo (or accept the user's description). Decide what sections the doc needs.

Read `references/doc-design-pass.md` in full — it has the section taxonomy (architecture overview, data flow, state and persistence, deployment, failure modes, glossary, where-to-start, etc.), the decision rules (which sections this kind of repo needs), and the format for the doc plan.

Default doc shape: headline + architecture overview + where-to-start. Add sections per the decision rules. Cap at 8–10 sections — beyond that, the doc becomes a documentation site rather than a single explainer.

Produce the doc plan as a short prose block. Show it to the user and ask for a quick sanity check before generation.

If the doc plan has more than 3 sections, run the doc-panel critique on the plan: spawn a subagent that reads `references/doc-panel-prompt.md` and returns a JSON report on the doc composition. Apply revision-worthy issues (`missing-section`, `wrong-section-order`, `prose-diagram-mismatch-in-plan`, `not-grounded-section`) by adjusting the plan before generation; surface the rest in the doc-level summary.

If the doc has 3 or fewer sections, skip the doc-panel.

**Opt-out.** If the user signals they want a quick doc — phrases like "rough overview", "skip the design pass", "just sketch it", "no critique" — drop to a minimal three-section doc (headline + one diagram set + where-to-start) and skip the doc-panel. The inner diagram procedure's syntax linter still runs (rendering correctness is non-negotiable).

### Phase B — Per-section generation

For each section in the plan, generate it. Section types:

- **Prose section.** Write the prose. Ground every claim in specific files / functions in the codebase. Cite paths (`src/foo.py:42`, `lib/bar.ts`). Don't fabricate. If the source isn't clear from your reading, surface it as a `TODO: confirm in <file>` rather than inventing a plausible-sounding answer. The prose's job is to explain *intent and structure*, not to repeat what the code already says.

- **Diagram-set section.** Follow the procedure in `references/diagram-procedure.md`. The diagram procedure has its own design pass (which diagrams in the set, how they relate), per-diagram generation, panel critique, syntax linter, and bounded revision. The output is the diagram set with its panel summaries, ready to embed in the doc section.

- **Hybrid section.** Prose + diagram set. The prose introduces or follows the diagram(s); both ground in the same source files. Write the prose first (so the diagrams have something to anchor to), then generate the diagrams.

Sections can be regenerated independently if the doc-panel flags one as broken without affecting the others.

### Phase C — Doc-level review

After all sections are generated and assembled, spawn a subagent that reads `references/doc-panel-prompt.md` and reviews the assembled doc. The doc-level panel checks coherence — do sections agree? do pointers resolve? is anything missing across the whole? It does NOT re-check what the diagram-level panel already covered (that's the inner loop's job).

If the doc-panel returns revision-worthy issues, revise once. Bounded — do not loop. The revision regenerates only the affected sections, not the whole doc.

Skip Phase C if Phase A's doc-panel was skipped (3-or-fewer-section doc, or opt-out).

### Phase D — Output

Return the complete markdown file as one artifact. The user should be able to save it directly as `docs/architecture.md` (or wherever the repo's docs live) without further editing.

The doc-level panel summary goes at the very end of the doc, in a `<details>` block titled "Generation notes" so it doesn't clutter the doc itself but stays auditable.

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

- `references/diagram-procedure.md` — the inner loop for generating diagram sets (used by Phase B for diagram-set sections; usable standalone). Self-contained: includes its own design pass, per-diagram generation, panel critique, syntax linter, and bounded revision.
- `references/doc-design-pass.md` — section taxonomy and decision rules for Phase A (which sections does the doc need).
- `references/doc-panel-prompt.md` — doc-level critique used by Phase A (when N > 3) and Phase C.
- `references/diagram-set-design-pass.md` — within-section rules for deciding the diagram-set composition (used by the inner diagram procedure).
- `references/diagram-set-panel-prompt.md` — within-section diagram-set critique (used by the inner diagram procedure).
- `references/panel-prompt.md` — per-diagram four-panelist critique (used by the inner diagram procedure).
- `references/syntax-lint-prompt.md` — Mermaid parser-footgun checklist (used by the inner diagram procedure).
- `references/diagnostic-checklist.md` — diagram-level failure modes for iteration mode.
- `references/mermaid-patterns.md` — Mermaid idioms and tactical guidance.
