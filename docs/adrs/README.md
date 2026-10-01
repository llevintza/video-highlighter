# Architecture Decision Records (ADRs)

All ADRs below are **Proposed** — starting points for MVP bake-offs, not accepted lock-in.  
Deciders: Architect (draft); Leo to confirm after bake-off on real youth-game footage.  
Date baseline: **2026-10-01**. Prices and APIs: re-verify; brief snapshots ~2026-10-01.

| ADR | Title | Status | One-line recommendation (experiment) |
|-----|--------|--------|--------------------------------------|
| [0001](./0001-ingest-and-chunking.md) | Ingest and chunking | Proposed | FFmpeg proxy + fixed ~4–8 s chunks; own `(start_ms, end_ms)` |
| [0002](./0002-moment-detection-and-embeddings.md) | Moment detection and embeddings | Proposed | Bake-off TwelveLabs (H1) vs owned SigLIP2/InternVideo2 + batch captions (H2); roster + manual tags first |
| [0003](./0003-vector-and-metadata-store.md) | Vector and metadata store | Proposed | Postgres + pgvector (or Qdrant if filtered ANN demands it) |
| [0004](./0004-blob-storage.md) | Blob storage | Proposed | Originals + proxy on R2/B2 or NAS; clips on demand / cache |
| [0005](./0005-runtime-and-hosting.md) | Runtime and hosting | Proposed | Queue + GPU ingest worker (scale-to-zero) + small always-on query API |

## How to read

Each ADR uses the same sections: Context → Decision drivers → Options considered → Recommendation (MVP experiment — not lock-in) → Rationale → Consequences → Open questions → References.

## Related

- Overview: [../architecture.md](../architecture.md)
- Research brief: `/workspace/video-highlighter-research-brief.md`
- Product ideation README: [llevintza/video-highlighter](https://github.com/llevintza/video-highlighter) (hypothesis only)
