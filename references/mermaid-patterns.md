# Mermaid patterns for system diagrams

Specific idioms for translating an architecture trace into Mermaid. Read this when you're writing the source and unsure how to express something. Pattern-by-pattern, with a complete working example at the end.

## Renderer config

Always use elk. The default dagre renderer produces tangled layouts on real diagrams. For inline rendering in a widget or page:

```javascript
mermaid.initialize({
  startOnLoad: false,
  theme: 'base',
  flowchart: {
    defaultRenderer: 'elk',
    curve: 'basis',         // smooth curves on edges
    nodeSpacing: 50,        // horizontal gap between sibling nodes
    rankSpacing: 60,        // vertical gap between layout ranks
  },
  fontFamily: 'system-ui, sans-serif',
  themeVariables: {
    fontSize: '13px',
    lineColor: '#73726c',
    textColor: '#3d3d3a',
    primaryColor: 'transparent',
    primaryBorderColor: '#888780',
  },
});
```

Mermaid v11 from `https://esm.sh/mermaid@11/dist/mermaid.esm.min.mjs` works in modern environments. For GitHub READMEs, you cannot configure elk — GitHub uses dagre. Warn the user; the diagram will still render but routing will be worse. See § "Publishing to GitHub README" below for specific dagre-survival constraints.

Direction: `flowchart LR` for request traces (left-to-right reads as time). `flowchart TB` for hierarchies. Pick one per diagram.

## Publishing to GitHub README

GitHub renders Mermaid with dagre, not elk. Diagrams that look great in [Mermaid Live](https://mermaid.live) with elk enabled can look genuinely bad on GitHub — overlapping nodes, edges crossing through labels, hexagons crammed against subgraph borders. Apply these constraints before shipping a diagram to a GitHub README, repo doc, or PR description.

The constraints aren't optional polish; they're the difference between "renders" and "renders well":

1. **Prefer simple shapes.** Stick to `[Box]`, `[(Cylinder)]`, and `(["Stadium"])`. Avoid `{{Hexagon}}` (dagre routes around them poorly and they often get clipped against subgraph borders), `[/Parallelogram/]` (parser-touchy with quoted labels), and `>Asymmetric]` (rarely renders cleanly). If the locked visual vocabulary calls for a hexagon, consider whether a labeled box would carry the same meaning.

2. **Cap subgraphs at 2–3 per diagram.** Each subgraph adds a constraint to dagre's solver; routing degrades fast above 3. Four subgraphs in a `flowchart LR` is the canonical "edges crossing through subgraph titles" failure on GitHub.

3. **Avoid `direction` overrides inside subgraphs.** Mixing `flowchart LR` with internal `direction TB` is well-supported by elk and produces crossed edges in dagre. Pick one direction at the diagram level and stick with it.

4. **Keep node labels under 3 lines.** `<br/>` heavy labels confuse dagre's node-sizing — the nodes get sized for the widest line and routing assumes that width, but the rendered layout doesn't always match. Two `<br/>` per label is the practical limit on GitHub.

5. **Edge labels under ~15 characters.** Dagre doesn't reflow long edge labels; they overlap nodes and adjacent labels. If you need a long label, abbreviate it and add a NOTE below the diagram, or split into two edges.

6. **6–12 nodes is the sweet spot.** Smaller diagrams compose better in dagre. Above ~15 nodes, the layout gets cramped on the typical GitHub-rendered width. Use sibling diagrams (see `references/diagram-set-design-pass.md`) instead of one large diagram.

7. **Test the actual GitHub render before committing.** Push to a draft PR or use a Markdown previewer that uses dagre. **Do not test only in Mermaid Live with elk** — that's a different renderer and will lie about the GitHub experience.

8. **For diagrams that must look good in both elk and dagre**, the intersection is "small and flat": no nested subgraphs, simple shapes, 6–12 nodes, short labels. If a diagram absolutely requires a layout that only elk handles well — 4+ subgraphs, heavy hexagonal transformation nodes, deeply nested groupings — declare it explicitly and ask the user where it'll be rendered. If the answer is GitHub, redesign for dagre or accept the suboptimal render with a note.

Two anti-patterns to recognize when reviewing a diagram bound for GitHub:

- **Multi-line edge labels.** `A -->|"part 1<br/>part 2"| B`. Renders fine in elk; in dagre the `<br/>` doesn't always create a line break in edge labels, so the label runs over the edge or off-screen.
- **Hexagon clusters.** Two or more `{{...}}` nodes adjacent to each other in a subgraph. Dagre tries to route the hexagons' edges around their angled corners and produces visible overlap.

The cleanest diagrams on GitHub are flat flowcharts with 8 or so nodes, 1–2 subgraphs, basic shapes, and short edge labels. If you find yourself reaching for hexagons or 4 subgraphs, that's a sign the diagram is doing too much — split into siblings.

## Subgraphs for semantic grouping

When you've named a semantic axis (trust boundary, control plane, sync/async), use `subgraph` to group nodes. The leading and trailing spaces in the label are intentional — they create breathing room around the title.

```
subgraph TENANT [" Tenant-scoped "]
  direction TB
  gw[Tenant gateway]
  skills[Skill registry]
end
```

Tint the subgraph fill with a low-alpha hex (the `22` suffix is alpha 0x22 ≈ 13% opacity):

```
style TENANT fill:#FAEEDA22,stroke:#BA7517,stroke-width:1px,color:#854F0B
```

Use 2-4 subgraphs maximum. More than that and the subgraphs themselves become visual noise.

## classDef for node category

`classDef` is the cleanest way to apply consistent styling to nodes that share a category. Define once, apply with `class`:

```
classDef tenant  fill:#FAEEDA,stroke:#854F0B,stroke-width:1px,color:#412402
classDef shared  fill:#F1EFE8,stroke:#5F5E5A,stroke-width:1px,color:#2C2C2A
classDef store   fill:#E1F5EE,stroke:#0F6E56,stroke-width:1px,color:#04342C

class gw,skills tenant
class plan,exec,llm,tools shared
class audit,rag store
```

Use the same color family inside a subgraph and as the node's classDef fill — the subgraph is the dimmer/tinted version, the nodes are the saturated version. This makes the grouping read clearly.

When picking colors, use established palettes (Tailwind, Material) rather than improvising hex values. Keep contrast between fill and text high; use the darkest stop of the same ramp for the text color.

## Node shapes carry meaning

Mermaid's node shapes are semantic, not decorative. Use them consistently:

- `[Box]` — generic component / service
- `{Decision}` — branch point, often a verifier or router. Use this when the architectural choice is "which way does this go?"
- `[(Cylinder)]` — persistent storage (database, audit log, vector store)
- `([Pill])` — external actor (user, third-party API). Marks "this is outside the system."
- `[/Parallelogram/]` — input/output, sometimes used for data files

Don't mix shape conventions within a diagram. If you start using cylinders for storage, every storage node uses a cylinder.

## Edges encode flow type

Three edge styles, three meanings:

- `A -->|label| B` — solid arrow, the main request flow. Solid means "this is part of the trace."
- `A -.->|label| B` — dotted arrow, side effects, lookups, audit writes, async signals. Dotted means "this happens, but it's not the main story."
- `A <-->|label| B` — bidirectional, request/response pairs (API calls, LLM I/O). Use sparingly; usually two separate labeled arrows are clearer.

Always label edges that cross subgraph boundaries. Inside a subgraph, labels are optional if the source-and-target make the meaning obvious.

## Architectural choices made visible

When the source material has an architectural decision worth surfacing, don't bury it in a subtitle. Make it structural.

**Tier selection** (e.g. "rules first, LLM fallback"):
```
verify{Verify}
verify -->|rules confident ≥0.8| accept[Accept]
verify -->|rules uncertain| llm_verify[LLM verifier]
llm_verify --> accept
```

**Lazy loading** (e.g. "metadata always, full body on demand"):
```
plan -->|always| skills_meta[Skill metadata]
plan -.->|on demand| skills_full[Skill bodies]
```

The dotted arrow signals "this only happens sometimes" without needing a paragraph of explanation.

**Cost-aware routing**:
```
router{LLM router}
router -->|fast / cheap| small[Small model]
router -->|slow / accurate| large[Large model]
```

## Cross-cutting concerns on edges, not as nodes

PII redaction, auth checks, rate limiting, observability — these intercept the flow. They go on edges as labels:

```
gw -->|"redacted prompt + tenant_id"| plan
exec <-->|"PII gate · model I/O"| llm
```

Or as a single sink with multiple dotted incoming arrows:

```
plan  -.->|audit| audit
exec  -.->|audit| audit
verify -.->|audit| audit
gw    -.->|audit| audit
```

Don't draw the audit log being written to four separate times in four separate places — one node, four arrows in.

## Syntax footguns the parser silently or noisily punishes

These are gotchas in Mermaid's parser that bite when you write the source by hand. They're not layout problems — they're "the diagram won't compile" or "the diagram compiles but renders something different than you wrote."

**1. Numbered-step labels parse as markdown lists.**

```
A[1. Plan] --> B[2. Execute]    ← breaks: "Unsupported markdown: list"
```

The space after the period triggers Mermaid's markdown-list detector inside the node label. Workarounds:

```
A[1.Plan] --> B[2.Execute]      ← drop the space
A[Step 1: Plan] --> B[Step 2: Execute]
A[① Plan] --> B[② Execute]      ← circled numbers
```

If a label is more than a couple words and has structured numbering, prefer the `Step N:` form — readers parse it faster than circled glyphs.

**2. Subgraph names with spaces need an ID + display-name pair.**

```
subgraph My Platform           ← breaks or renders unexpectedly
  ...
end

subgraph platform [" My Platform "]   ← correct: ID, then quoted display
  ...
end
```

The leading/trailing spaces inside the quoted label give the title visual breathing room. Reference the subgraph by its ID (`platform`) in edges, not by its display name.

**3. Parentheses and quotes inside node labels.**

```
A[Plan (with retries)]          ← parens confuse the parser
A["Plan (with retries)"]        ← quoting the whole label fixes it
A[Plan · with retries]          ← or use middle-dot · / em-dash — / pipe |
```

Same with quotes inside labels: wrap the whole label in double-quotes and escape inner ones, or substitute typographic quotes (`"`/`"`).

**4. `<br/>` for line breaks works inside any node shape, but only inside quoted labels for some shapes.** When in doubt, quote the label:

```
A["Line one<br/>line two"]
```

**5. `@` characters in labels parse as edge IDs (Mermaid v11+).**

Mermaid v11 introduced an `e1@-->` syntax for naming edges. Any bare `@` inside a node label trips this parser, even when there's no edge nearby:

```
metrics[(Metrics<br/>recall@k)]      ← breaks: "got LINK_ID"
metrics[("Metrics<br/>recall@k")]   ← quote the label, fixes it
```

GitHub renders Mermaid with v11, so this bites diagrams that worked locally on v10. Quote any label containing `@`, `recall@k`, `p@N`, email addresses, or decorator-style syntax.

**6. Edge labels with special characters need quoting too.**

```
A -->|cost > 0.8| B             ← breaks on >
A -->|"cost > 0.8"| B           ← quote the label
```

The bracket characters `[]`, `()`, `{}` are especially treacherous in unquoted edge labels because the parser sees them as the start of a node-shape declaration. `A -.->|products[]| B` will fail with a "got 'SQS'" parse error on `[`, even though to a human it's obviously a label. Always quote labels containing brackets, parens, or braces:

```
A -.->|products[]| B            ← parser thinks `[` opens a new node label, errors
A -.->|"products[]"| B          ← quote the label, fixes it
```

This bites GitHub-rendered diagrams especially hard because the live editor's error message points at the right line but the GitHub renderer just shows "Unable to render rich display."

**7. Reserved-ish keywords as node IDs.** `end`, `class`, `style`, `subgraph`, `direction` — don't use these as node IDs. They'll either error or silently get interpreted as a directive. Pick `end_node`, `cls`, etc.

**8. Quality checklist before pasting.** Quick scan before you ship the source:

- No `[N. ...]` patterns inside node labels (markdown-list trap)
- All subgraphs declared as `subgraph id ["Display Name"]`
- All edges reference subgraphs / nodes by ID, not display name
- Labels with spaces, parens `()`, square brackets `[]`, braces `{}`, quotes, `@`, or `>` are wrapped in `"..."`
- `linkStyle` indices recounted if you added or removed any edge

Mermaid's error messages are usually accurate but can be terse — when in doubt, paste into Mermaid Live with the elk renderer enabled; the live editor's error highlighter pins the offending line.

## linkStyle: the indexing footgun

`linkStyle N stroke:...` colors the Nth edge in declaration order, counting from 0. The footgun: if you add or remove an edge, every linkStyle index after it shifts. This silently breaks color coding.

Two options:

1. **Put linkStyle at the very end of the source**, after all edges are declared, and recount carefully whenever you edit.
2. **Use classDef-equivalent for edges** — define a class on the edge directly. Mermaid v10+ supports this:

```
classDef auditEdge stroke:#1D9E75,stroke-width:1.5px
plan  -.->|audit| audit:::auditEdge
```

Wait — that syntax applies to the *node* on the right, not the edge. For edge classes, the cleanest approach is still numbered linkStyle, just kept at the bottom of the file with a comment counting which edges are which.

If linkStyles render incorrectly (wrong arrow colored or no effect), it's almost always an off-by-one in the counting. Mermaid silently no-ops invalid indices, so the diagram looks plausible but is wrong.

## Standalone HTML render template

When the user wants a self-contained HTML page that renders the Mermaid:

```html
<div id="diagram"></div>
<script type="module">
import mermaid from 'https://esm.sh/mermaid@11/dist/mermaid.esm.min.mjs';
const dark = matchMedia('(prefers-color-scheme: dark)').matches;
await document.fonts.ready;

mermaid.initialize({
  startOnLoad: false,
  theme: 'base',
  flowchart: { defaultRenderer: 'elk', curve: 'basis', nodeSpacing: 50, rankSpacing: 60 },
  themeVariables: {
    darkMode: dark,
    fontSize: '13px',
    lineColor: dark ? '#9c9a92' : '#73726c',
    textColor: dark ? '#c2c0b6' : '#3d3d3a',
  },
});

const src = `flowchart LR
  %% your diagram source here
`;

const { svg } = await mermaid.render('diagram-svg', src);
document.getElementById('diagram').innerHTML = svg;
</script>
```

Three details that matter:
- `await document.fonts.ready` before render — Mermaid measures text width during layout; if fonts haven't loaded, boxes are sized for the wrong font and clip after the real font swaps in.
- `dark` media query for theme — Mermaid handles dark mode if you pass `darkMode: true` in theme variables.
- `startOnLoad: false` plus explicit `mermaid.render()` — gives you control over when rendering happens, important if the diagram source is built dynamically.

## Complete working example

A multi-tenant agent platform: request trace with three trust boundaries, a plan-execute-verify loop, audit sink, and cross-run feedback channel.

```
flowchart LR
  user([User])

  subgraph TENANT [" Tenant-scoped "]
    direction TB
    gw[Tenant gateway]
    skills[Skill registry]
  end

  subgraph CORE [" Shared platform — orchestrator "]
    direction TB
    plan[Plan]
    exec[Execute]
    verify{Verify}
    adjust[Adjust]
    llm[LLM router]
    tools[Tool adapters]
    feedback[Feedback pipeline]
  end

  subgraph STORE [" Per-tenant storage "]
    audit[(Audit log)]
    rag[(Vector store)]
  end

  user -->|raw request| gw
  gw -->|"redacted prompt + tenant_id"| plan
  plan -.->|capabilities lookup| skills
  skills -.->|skill subset| plan
  plan --> exec
  exec <-->|"PII · model I/O"| llm
  exec --> tools
  tools -.-> rag
  rag -.-> tools
  tools --> exec
  exec --> verify
  verify -->|fail| adjust
  adjust -->|retry| plan
  verify -->|pass| gw
  gw -->|response| user

  verify -->|verdict| feedback
  feedback -.->|policy update| skills

  plan  -.->|audit| audit
  exec  -.->|audit| audit
  verify -.->|audit| audit
  gw    -.->|audit| audit

  classDef tenant fill:#FAEEDA,stroke:#854F0B,color:#412402
  classDef shared fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
  classDef store  fill:#E1F5EE,stroke:#0F6E56,color:#04342C

  class gw,skills tenant
  class plan,exec,adjust,llm,tools,feedback shared
  class verify shared
  class audit,rag store

  style TENANT fill:#FAEEDA22,stroke:#BA7517
  style CORE   fill:#F1EFE822,stroke:#888780
  style STORE  fill:#E1F5EE22,stroke:#1D9E75
```

This is the shape to aim for: subgraphs grouping by trust boundary, classDef coloring nodes by category, edge styles distinguishing main flow from side effects, audit as a single sink, the plan-execute-verify loop visible as a cycle, the cross-run feedback as a separate dotted edge to a different layer.
