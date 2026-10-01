# Video Highlight Creator

## Overview

**Video Highlight Creator** is a platform that turns full-length sports game recordings (starting with basketball) into a searchable library of moments.  

Instead of re-processing the raw video every time someone wants a highlight reel, the system runs a **single upfront embedding pipeline**. The entire game is broken into time-based chunks, vectorized, and stored in a vector database with rich metadata.  

After that, any user can generate custom highlight reels on demand — by player, by play type (defensive, offensive, specific actions), by game, or any combination — simply by querying the embeddings.

This design eliminates redundant heavy compute while giving parents, coaches, and players flexible, personalized access to the footage.

## Documentation

Working drafts for the MVP experiment live under [`docs/`](docs/README.md):

- [Product requirements (PRD)](docs/prd.md)
- [Architecture overview](docs/architecture.md)
- [Architecture Decision Records](docs/adrs/) (ingest, embeddings, vector store, blobs, runtime)

Stack choices in those docs are **Proposed** bake-off starting points (options → recommendation → open questions), not locked production decisions.

## Core Concept

1. **Ingest once** — Upload a full game video.
2. **Chunk + Embed** — Split the video into meaningful segments (scene changes, fixed intervals, or action boundaries). Generate multimodal embeddings (visual + optional audio/context) that capture:
   - What is happening (shot, steal, rebound, pass, etc.)
   - Who is involved (player identification via jersey, face, or tracking)
   - Game context (score, period, location on court when possible)
3. **Index** — Store the embeddings + metadata (timestamps, game ID, player tags, play type labels) in a vector database.
4. **Query on demand** — Users express what they want in natural language or structured filters. The system retrieves the most relevant segments and assembles them into a highlight clip.
5. **Export** — Deliver a polished video reel (or just the timestamps) without ever re-analyzing the original footage.

## High-Level Architecture

```
[Raw Game Video]
       │
       ▼
┌─────────────────────┐
│  Ingestion Pipeline │  ← FFmpeg / scene detection / chunking
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Embedding Service   │  ← Multimodal model (e.g. CLIP-family, video-native, or sports-tuned)
│ (visual + audio)    │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Vector Database     │  ← Embeddings + metadata (timestamps, players, play type, game ID)
│ + Metadata Store    │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Query / Retrieval   │  ← Semantic search + filters (player, play type, game, etc.)
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Highlight Assembler │  ← Stitch matching segments → export clip
└─────────────────────┘
```

## Key Requirements (Captured from Discussion)

- **One-time heavy processing** — Embed the entire video only once.
- **Per-user, on-demand configuration** — Parents can filter by their child; coaches can pull defensive stops, offensive sets, specific play types, or whole-game reviews.
- **Flexible querying** — Support combinations of:
  - Specific player
  - Play type (defensive highlights, offensive highlights, etc.)
  - Specific game or date range
  - Free-form natural language (“show me all steals by #12 in the second half”)
- **Avoid redundant compute** — Never re-process the full video for each new highlight request.
- **Scalable for multiple users** — One shared embedding index serves many concurrent custom requests.
- **Starting domain** — Basketball games (youth / high-school / club level recordings of the user’s daughter’s games). Designed to be extendable to other sports later.

## Benefits

- Dramatically lower ongoing compute cost.
- Instant highlight generation after the initial embedding pass.
- Highly personalized experience for different stakeholders (parents vs coaches vs players).
- Future-proof: new query types or improved models can leverage the same stored embeddings.

## Suggested Tech Stack (Boilerplate / Starting Point)

These are recommendations only — final choices can evolve:

| Layer              | Options                                      | Notes |
|--------------------|----------------------------------------------|-------|
| Video processing   | FFmpeg + PySceneDetect                       | Chunking & scene detection |
| Embedding models   | CLIP / OpenCLIP / SigLIP / TwelveLabs / InternVideo2 / sports-tuned models | Multimodal preferred |
| Vector database    | Qdrant, Weaviate, Pinecone, Milvus, or FAISS | Hybrid search (vector + metadata filters) |
| Backend            | Python + FastAPI                             | Simple, async-friendly |
| Storage            | S3-compatible object storage                 | Raw videos + generated clips |
| Frontend (later)   | Next.js or simple React                      | Upload + query + player interface |

## Project Status

Early ideation / architecture definition stage.  
This README captures the high-level vision and requirements from the initial conversation.

## Next Steps (Suggested)

1. Decide on concrete embedding model and vector DB for the MVP.
2. Prototype the ingestion + embedding pipeline on a single sample game video.
3. Build a simple query interface that returns timestamps + short clips.
4. Add basic player identification (jersey number OCR or face recognition).
5. Expand to multi-user access and richer play-type taxonomy.

---

*Generated from the “Video Highlight Creator” brainstorming session.*
