# recs_two_tower — two-tower recommendation system

A worked example of the skill applied to a Keras-based two-tower deep-learning recommendation system. Offline training writes a model + preprocessor + item-embedding artifacts; online serving loads them and answers `recommend(user_features, top_k)` calls via a user-tower forward pass plus ANN search (FAISS or sklearn fallback) over the precomputed item embeddings.

**Design summary (step 0):** N=1, single-diagram set. Step 0 considered a 2-diagram set (offline training as a lifecycle sibling) but settled on N=1 because the offline pipeline appears here only as an artifact-write source, not in detail — the offline subgraph has 2 nodes and the artifact handoff is the load-bearing structural boundary. A 2-diagram split would add a 6-8 node training pipeline diagram on the side, which is the right call *if* the user is asking about training; for the canonical "how does serving work" request, single-diagram with the handoff visible is more legible. No design-panel critique runs for N=1. The user can request a "training pipeline" sibling as a follow-up.

The diagram exercises the **request-trace-trust-bounds** archetype with the Platform / SRE engineer SME persona. Step 6 revised the original peer-positioned `ARTIFACTS` subgraph to fold inside `SERVING` as "loaded at construction" inputs, which makes the actual two-surface (offline / online) lifecycle axis visible — artifacts are the handoff *between* surfaces, not a third surface of their own.

## Plan

- **Concrete entry point.** A signed-in homeowner calls `service.recommend(user_features, top_k=10)` — a single online recommendation request through the `LocalInferenceEngine`.
- **Ordered path.** caller → `InferenceManager` → `LocalInferenceEngine` → `_preprocess_user_features` (vocab/encoder lookup) → `UserTower` forward (embeds + LSTM + dense + L2 normalize) → ANN search (FAISS or sklearn fallback) over precomputed item embeddings → post-filter (`exclude_items`, idx → `item_id`, top_k) → return `List[(item_id, score)]`. Plus offline → online artifact handoff (model + preprocessor + item embeddings written by training, loaded at engine construction).
- **Semantic axis.** Trust / lifecycle boundary — offline build vs online serving. Artifacts are not a third zone; they are the handoff surface, drawn as loaded-at-construction inputs inside `SERVING` with offline-write arrows crossing the lifecycle boundary.
- **Out of scope.** Training pipeline internals, Vertex AI deployment path, optimized graph-mode variant, failure / cold-start paths.

## Mermaid source

```mermaid
flowchart LR
  caller([Caller<br/>run_demo / API client])

  subgraph OFFLINE [" Offline (built once per training run) "]
    direction TB
    trainer[TwoTowerTrainer<br/>training.py]
    extract[extract_embeddings<br/>item tower over all items]
  end

  subgraph SERVING [" Online serving runtime "]
    direction TB
    mgr[InferenceManager<br/>mode=local]
    engine[LocalInferenceEngine<br/>recommend_items]
    pre[_preprocess_user_features<br/>vocab + encoder lookup]
    user_tower[UserTower forward<br/>embeds + LSTM + dense + L2]
    ann{ANN backend?}
    faiss[FAISS IndexFlatIP<br/>inner-product search]
    sk[sklearn NearestNeighbors<br/>cosine fallback]
    post[Post-filter<br/>exclude_items + idx to item_id + top_k]

    pre_art[("preprocessor.json<br/>loaded at construction")]
    model_art[("final_model.keras<br/>loaded at construction")]
    item_emb[("item_embeddings.npy + item_ids.npy<br/>loaded at construction")]
  end

  caller -->|"user_features dict, top_k=10"| mgr
  mgr -->|delegate| engine
  engine -->|raw features| pre
  pre -.->|lookup| pre_art
  pre -->|"int-encoded tensors<br/>+ padded item_sequence"| user_tower
  user_tower -.->|weights| model_art
  user_tower -->|"L2-normalized<br/>user embedding (1x64)"| ann
  ann -->|"FAISS available<br/>(built at load)"| faiss
  ann -.->|"FAISS missing<br/>fallback path"| sk
  faiss -.->|read| item_emb
  sk -.->|read| item_emb
  faiss -->|"(scores, idx) top-2k"| post
  sk -->|"(distances, idx) top-2k"| post
  post -->|"List[(item_id, score)] top-k"| engine
  engine -->|recommendations| mgr
  mgr -->|response| caller

  trainer -.->|"writes (offline → online handoff)"| model_art
  trainer -.->|"writes (offline → online handoff)"| pre_art
  trainer --> extract
  extract -.->|"writes (offline → online handoff)"| item_emb

  classDef offline fill:#EAE4F2,stroke:#5B3A8C,color:#2A1850
  classDef serving fill:#FAEEDA,stroke:#854F0B,color:#412402
  classDef store   fill:#E1F5EE,stroke:#0F6E56,color:#04342C
  classDef ext     fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A

  class trainer,extract offline
  class mgr,engine,pre,user_tower,faiss,sk,post,ann serving
  class model_art,pre_art,item_emb store
  class caller ext

  style OFFLINE fill:#EAE4F222,stroke:#5B3A8C
  style SERVING fill:#FAEEDA22,stroke:#BA7517
```

## Notes

- **Architectural choices surfaced visually.** The ANN backend selection (FAISS primary vs sklearn fallback) is a decision diamond fanning out to two branches, with the fallback drawn dotted. The artifact handoff is dotted writes from `OFFLINE` crossing into `SERVING` with explicit `(offline → online handoff)` labels — making the lifecycle boundary the actual axis the diagram encodes.
- **Renderer config.** `flowchart LR`, elk renderer, `curve: 'basis'`, `nodeSpacing: 50`, `rankSpacing: 60`. Subgraph fills use the `22` alpha suffix; classDef fills are the saturated version of the same hue family. On GitHub (dagre), routing will be looser but readable.

## Panel summary (from step 6)

Panel revised one issue: `wrong-trust-surface` (the `ARTIFACTS` subgraph was peer-positioned with `OFFLINE` and `SERVING`, but artifacts are the handoff surface, not a third lifecycle zone — folded into `SERVING` with explicit offline-write arrows crossing the boundary). Four borderline issues surfaced: `noun-inventory` at 14 nodes (soft band), `choices-buried` on `L2 normalize` (which is what makes FAISS IndexFlatIP equal cosine similarity), and two `inconsistent-edge-style` calls on the FAISS/sklearn branching.
