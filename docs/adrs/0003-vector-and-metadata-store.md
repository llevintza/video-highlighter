# ADR-0003: Vector and metadata store

- Status: Proposed
- Date: 2026-10-01
- Deciders: Architect (draft); Leo to confirm after bake-off

## Context

Queries are **hybrid**: approximate nearest neighbor (ANN) over embeddings **plus** filters on `player`, `game_id`, `play_type`, `period`, `time_range`, and confidence. Early scale: tens–hundreds of games × ~1k–3k chunks ≈ **~10k–1M vectors**—well below billion-scale systems.

The store must support the success gate (&lt;5 s useful clip list from index, no full-video LLM) and remain portable when embedding models change (`model_version`).

**Stack is not locked.** Postgres+pgvector vs Qdrant (vs others) is an MVP experiment choice, revisitable after filter/latency bake-off.

## Decision drivers

- Filtered ANN that works for “#12 steals in second half of game X”
- Low ops burden for a family → small-club product
- Cost dominated by instance size, not surprise vector SKUs, at early scale
- OSS or exportable exit (avoid irreversible lock-in)
- One place for roster, jobs, and tags if possible—or a clear split with sync rules
- Reindex when models change

## Options considered

### A. pgvector on Postgres (managed: Neon / RDS / Cloud SQL / Supabase, or self-host)

- Pros: one database for app metadata + vectors; SQL filters natural; cheapest ops if Postgres is needed anyway; fine under ~millions of vectors for this product’s early scale.
- Cons: hybrid lexical+vector is DIY (FTS + vector); filtered ANN needs care (indexes, iterative scan settings); not purpose-built for extreme QPS.
- Cost: dominated by instance size, not a separate vector SKU.

### B. Qdrant (OSS self-host or Qdrant Cloud)

- Pros: strong **filtered HNSW**; sparse+dense hybrid; payload indexes; Apache-2.0 escape hatch. Free Cloud tier for prototypes (0.5 vCPU / 1 GB RAM / 4 GB disk).
- Cons: another system if self-hosting; Cloud billed on resources.
- Fit: excellent when metadata filters are first-class (player + play type + game).

### C. Weaviate (OSS or Weaviate Cloud)

- Pros: mature **native hybrid (BM25 + vector)**; modules ecosystem; Cloud from ~$45/mo Flex + dimension/storage meters (~2026-10-01).
- Cons: more moving parts than pgvector; Cloud cost floor.
- Fit: strong if NL queries benefit from keyword+semantic fusion (play names, jersey numbers as tokens).

### D. Pinecone (managed serverless)

- Pros: zero-ops; metadata filters; serverless storage + RU/WU pricing.
- Cons: cost grows with query/storage; hybrid/sparse story less convenient than Weaviate; no self-host.
- Fit: fine for MVP if ops avoidance outweighs cost predictability.

### E. Milvus / Zilliz

- Pros: scales to huge corpora; hybrid in recent versions.
- Cons: ops-heavy self-host; overkill at &lt;1M vectors.
- Fit: defer unless multi-tenant sports org scale appears.

### F. FAISS / LanceDB / Chroma (embedded)

- Pros: fast local prototypes; LanceDB nice for disk-backed multimodal tables.
- Cons: still need durable metadata, auth, backup, multi-user API.
- Fit: local experiments only—not production multi-parent access without a real data plane.

### Tradeoff snapshot

| Store | Hybrid filter strength | Ops | Cost at ~100k–1M vec | Portability |
|---|---|---|---|---|
| pgvector | Good (SQL) / DIY lexical | Low if Postgres exists | Low | High |
| Qdrant | Excellent filters | Low–med | Low–med | High (OSS) |
| Weaviate | Excellent hybrid | Low–med | Med (Cloud floor) | High (OSS) |
| Pinecone | Good filters | Lowest | Med–high w/ QPS | Low |
| Milvus | Excellent at scale | High | Low self-host / med Zilliz | High |
| FAISS/etc. | DIY | High for prod | Low $ / high labor | N/A |

## Recommendation (MVP experiment — not lock-in)

**For MVP: Postgres + pgvector** for game/clip/roster metadata and vectors in one place, **or Qdrant** if bake-offs show filtered ANN latency/recall pain in Postgres. Either choice keeps an OSS exit. Revisit **Weaviate** if hybrid BM25 proves critical for jersey numbers and play jargon. Avoid **Milvus** until scale demands it. Use FAISS/LanceDB/Chroma for laptop experiments only.

Illustrative schema sketch (**not prescribed**):

```text
chunks(id, game_id, start_ms, end_ms, embedding,
       play_types[], player_numbers[], ocr_text,
       model_version, confidence)
```

Always version embeddings (`model_version`) so H1↔H2 or model upgrades can reindex without orphaning metadata.

## Rationale

Early vector counts do not justify heavy vector platforms. SQL-native filters match how coaches think (game, period, jersey). Qdrant remains the escape hatch if filtered HNSW quality is the bottleneck. Portability matters for youth athlete data and for swapping H1 TwelveLabs-derived vectors vs H2 owned embeddings.

## Consequences

**Positive**

- Simple ops story if Postgres is already planned for jobs/roster
- Clear path to &lt;5 s queries at this scale when embeddings are precomputed
- Reindex strategy is feasible

**Negative / risks**

- Underestimating filtered ANN tuning in pgvector
- Dual-writing metadata if Qdrant is added later without a sync plan
- Pinecone-like managed comfort can hide long-term cost if QPS spikes after games

**Neutral**

- Multi-tenant isolation (many families) may later require row-level security or separate collections—out of MVP scope

## Open questions

1. Prefer “one database” simplicity or best-of-breed vector engine?
2. Multi-tenant isolation needed soon (many families) or single-family first?
3. Must embeddings be exportable/reindexable when models change (versioning strategy confirmed)?
4. Will H1 TwelveLabs remain the query engine for some games while H2 uses local vectors (dual-read complexity)?

## References

- Research brief §3 — `/workspace/video-highlighter-research-brief.md`
- [Qdrant pricing](https://qdrant.tech/pricing/)
- [Weaviate pricing](https://weaviate.io/pricing)
- [Vector DB comparison 2026 (Semantic.io)](https://semantic.io/insights/vector-database-comparison-2026)
- [Vector DB pricing comparison (Cipher Projects)](https://www.cipherprojects.com/blog/posts/vector-database-pricing-comparison-2026/)
- Prices / comparisons: snapshots ~2026-10-01
