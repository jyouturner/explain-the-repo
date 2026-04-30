# Skill-building checklist

A scannable list of "did you…?" questions for shipping a Claude Code skill. Each item is one question + one sentence on why it matters + a pointer to the failure mode that produced it.

This is not a process — it's a filter. Most items are obvious for simple skills; they exist to catch the non-obvious cases that bit us during the journey documented in [`tutorial.md`](tutorial.md).

---

## Before you start designing

**Q1 — Is this a skill, or could it be a tool / hook / slash command?** Skills are markdown that the model reads at runtime; they're best for *guidance* the model needs to follow, not for *automation*. If the answer is "execute X every time the user does Y," a hook or a custom command is probably the right shape, not a skill. *Source: skill-vs-mechanism distinction Claude Code's docs already cover.*

**Q2 — What's the user's actual deliverable?** Write it down before designing the procedure. The first deliverable you write down is often wrong; the artifact that actually helps the user is usually one layer up. (We started at "draw a Mermaid diagram" and the deliverable was "an architecture document containing diagrams.") *Source: §5 of `lessons.md`.*

**Q3 — What's the failure mode you're correcting?** A skill that doesn't have a specific failure mode in mind tends to over-prescribe. Name the AI failure you're trying to prevent: noun-inventory diagrams, fabricated specifics, rambling prose, etc. The skill's procedure exists to prevent that.

---

## While you're building

**Q4 — Are any failure modes documented but not enforced?** Passive references (a "syntax footguns" doc, a "common pitfalls" section) get ignored at the moment of failure. If the failure mode is mechanical (a regex would catch it), it should be a procedural check (e.g. a syntax linter), not a doc the model is supposed to remember. *Source: §2 of `lessons.md`. The `products[]` parser bug shipped past a perfectly good "syntax footguns" reference because reading isn't enforcement.*

**Q5 — If you have multiple critic personas, does each have a non-overlapping rubric and an explicit "out of scope for you"?** "Channel many perspectives" only works when each perspective has a distinct lane. Without that, four panelists collapse to one generic critique. *Source: §1 of `lessons.md`.*

**Q6 — Does the model have an explicit "I don't know" path?** Tell the model what to do when it can't verify a claim — `mark TODO: confirm in <file>`, surface the gap to the user, refuse to assert. Without that, models fabricate confident-sounding fiction. *Source: §6 of `lessons.md`. This caught three real codebase inconsistencies across the cold-run validations.*

**Q7 — If you're adding a second or third critique pass, design the opt-out at the same time.** Token costs compound invisibly. Each step feels small; the total isn't. Add a documented "user signal that bypasses this" path the moment you add a layered critique, not after it gets expensive. *Source: §4 of `lessons.md`. We went from 1 model call per request to ~21 worst-case before adding the opt-out.*

**Q8 — Are filter / threshold rules calibrated against real artifacts?** The skill's *architecture* tends to be roughly right; the *thresholds* (when to flag, when to revise, when to fan out) are where real bugs live. Run the loop end-to-end on at least one realistic artifact before shipping. *Source: §3 of `lessons.md`. Five hand-runs surfaced the autoresearch reviser regression that no amount of design-doc iteration predicted.*

**Q9 — Are subagent / tool calls scoped narrowly?** A subagent that "reads the panel-prompt and applies the rubric" is testable. One that "thinks about the diagram" is not. Each subagent invocation should have a JSON schema for its output and a one-sentence description of what it does that fits in your head.

---

## Before you ship

**Q10 — Have you run the *whole* skill end-to-end against a real artifact, in a clean context?** Authoring-context grading is biased — the model that just wrote the artifact is grading its own work. Spawn a fresh subagent (or a fresh session) that loads the skill files cold and produces an output. Eyeball whether it actually works. *Source: §3 + the cold-run validations in `tutorial.md` Phase 8–9.*

**Q11 — Does your "trigger" description match what users actually type?** The skill description tells Claude when to invoke. Users don't say "produce architecture documentation"; they say "explain this repo." Test the description against three or four realistic phrasings. If it triggers on too few, broaden; if too many, narrow.

**Q12 — Have you tested the "should NOT trigger" cases?** Skills that fire when they shouldn't are worse than skills that don't fire when they should. Include a few "out of scope" prompts in evals and verify the skill steps aside (e.g. recommends `erDiagram` instead of forcing tables into a flowchart). *Source: `evals/evals.json` `out-of-scope-erd` case.*

**Q13 — Is there a worked example in the repo, produced by actually running the skill?** Hand-shaped examples that *describe* what the skill should produce drift from reality. A cold-run example proves the skill works, surfaces real failure modes, and gives readers a concrete artifact to evaluate. *Source: §3 + `examples/real-repos/` in this repo.*

**Q14 — What happens when the model's context budget runs low?** Long skills (multi-page SKILL.md, many references) compete with the user's actual task for context. Test with a non-trivial user prompt and see whether the skill drops procedure under pressure. If it does, slim down or move depth into reference files the model loads only when needed.

---

## After you ship

**Q15 — Are your design docs being kept current?** Static design docs rot in weeks. Live ones (updated with calibration findings, resolution notes, post-ship adjustments) are way more valuable — they answer "why is the skill shaped this way?" with evidence attached. *Source: §7 of `lessons.md`. The `design/` folder in this repo is the worked example.*

**Q16 — Do you have a way to know when the skill is failing?** Bad outputs from real users are the highest-signal feedback. Make the skill easy to file issues against (link in README, "if X happens, please report" instruction). *Source: `CONTRIBUTING.md`.*

**Q17 — Is the version / lineage clear?** Skills evolve. CHANGELOG entries that document *why* a change was made (not just what changed) give the next person — including future-you — the context to extend or revise without re-deriving the rationale.

---

## Summary card (for pasting into a wiki)

> **Before designing:** Is this actually a skill? What's the user's deliverable? What failure mode are you correcting?
>
> **While building:** Documented vs enforced? Disjoint critic rubrics? "I don't know" path? Opt-out alongside any layered critique? Filter thresholds calibrated against real artifacts? Narrowly-scoped subagent calls?
>
> **Before shipping:** Cold-run validation in a fresh context? Trigger phrases tested? Should-not-trigger cases tested? Cold-run worked example committed? Context budget tested?
>
> **After shipping:** Design docs kept current? Issue feedback path? Lineage in CHANGELOG?
