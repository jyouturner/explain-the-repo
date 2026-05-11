# explain-the-repo

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that produces architecture documentation for any codebase: prose sections grounded in your actual files, plus Mermaid diagrams generated through a procedure designed to catch the usual AI-doc failure modes (sprawling diagrams, fabricated specifics, missing return arrows, prose that doesn't match the diagram).

Status: experimental, v0.4.

## Examples

Real outputs from running the skill cold on four codebases:

- [`autoresearch.md`](examples/real-repos/autoresearch.md) — 7 sections, 4 diagrams (AI agent with multi-cadence loops and cross-run state).
- [`enterprise.md`](examples/real-repos/enterprise.md) — 7 sections, 4 diagrams (framework with two reference implementations).
- [`recs-two-tower.md`](examples/real-repos/recs-two-tower.md) — 6 sections including a Lifecycle section (offline-training / online-serving ML).
- [`graphql-node.md`](examples/real-repos/graphql-node.md) — 6 sections with a sequenceDiagram for data flow (small Lambda-fronted API).

Each doc has: a *where to start reading* section with file pointers, an architecture overview with 1–5 Mermaid diagrams, component summaries grounded in cited files, and (when warranted) lifecycle / data-flow / state-and-persistence / glossary sections. Honest `TODO: confirm` markers surface for things the skill couldn't verify, rather than fabricated answers.

## Install

```bash
git clone https://github.com/jyouturner/explain-the-repo.git \
  ~/.claude/skills/explain-the-repo
```

Claude Code auto-discovers skills under `~/.claude/skills/`. No config.

## Use it

In any project, prompt Claude:

> *"Explain this repo — architecture doc for a new engineer."*

That's it. The skill surfaces a doc plan first (sections it intends to produce), waits for your sanity-check, then generates the doc.

Variations that change cost or shape:

| Want | Phrase |
| --- | --- |
| Cheap, fast overview (skips the critique passes, ~3–4× less token spend) | *"Quick architecture overview, no critique."* |
| Just a diagram, no doc wrapper | *"Draw a diagram of the request flow through this service."* |
| Redraw an existing messy diagram | *"This diagram is messy — can you redraw it?"* (paste the source) |

## How it works (briefly)

Two-level procedure. Outer loop designs the doc (which sections, what's out of scope) and reviews the assembled output. Inner loop, invoked per diagram-set section, produces 1–5 sibling diagrams under a parallel three-subagent critique (two panel critics + one syntax linter). See [`SKILL.md`](SKILL.md) for the full procedure and [`design/`](design/) for the rationale.

## Limitations

- **Diagram zoom level.** C4 "container" only — services and components in one system. No multi-level (context → container → component → code). Use [Structurizr](https://structurizr.com/) for that.
- **Mermaid only.** No D2, PlantUML, Graphviz.
- **Token cost is real.** Full-doc invocations run 3–10× the tokens of a single-shot diagram. Use the opt-out path when it matters.
- **GitHub renders Mermaid with dagre, not elk.** Diagrams look cleaner in [Mermaid Live](https://mermaid.live) with elk than in a GitHub-rendered README.
- **Doc design pass is heuristic.** "Does this need a Lifecycle section?" calls are rules of thumb, not benchmarked. Real-world use will tune them.
- **Not benchmarked at scale.** Treat outputs as v1. Review before shipping anywhere important.

## Files

- [`SKILL.md`](SKILL.md) — the skill definition.
- [`references/`](references/) — the procedure files (doc design pass, inner diagram procedure, panel-critique prompts, syntax linter, Mermaid patterns and footguns).
- [`examples/real-repos/`](examples/real-repos/) — the cold-run worked examples linked above.
- [`examples/walkthrough.md`](examples/walkthrough.md) — step-by-step walk of the inner diagram procedure on a multi-tenant RAG service.
- [`examples/v0.5-walkthrough.md`](examples/v0.5-walkthrough.md) — end-to-end walk of the doc-level procedure (Phase 0 → E) on a real codebase. Shows the v0.5 mechanisms in action: in-session-author detection, existing-docs scan, visual-vocabulary lock-in, token-budget output, vocabulary conformance during diagram generation, the reader test with a real subagent JSON output, and the generation-notes block.
- [`evals/evals.json`](evals/evals.json) — test prompts.
- [`design/`](design/) — design rationale (panel-critique loop, integrated flow, calibration findings).
- [`docs/skill-building/`](docs/skill-building/) — three documents extracting the skill-building lessons from this project: a one-page operational checklist, a seven-insights essay, and a commit-by-commit tutorial. Useful if you're building skills of your own.
- [`CHANGELOG.md`](CHANGELOG.md) — version history including the v0.4 rename from `system-diagram-in-mermaid`.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Issues welcome — especially bad outputs (attach the prompt + the doc the skill produced) and archetypes the skill mishandles.

## License

MIT — see [`LICENSE`](LICENSE).
