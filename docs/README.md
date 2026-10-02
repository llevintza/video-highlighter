# Video Highlighter — Architecture Docs

Working architecture notes for Leo’s youth/HS/club basketball **video highlight** product.  
**Source of truth for options and cost sketches:** `/workspace/video-highlighter-research-brief.md` (prepared 2026-10-01 EDT).  
Vendor prices cited in these docs are **~2026-10-01 snapshots** — re-verify before budgeting.

> **Stack is not locked.** Every ADR and the overview frame *options → MVP experiment recommendation → rationale → consequences → open questions*. Choices are bake-off starting points, not fiat. CoS GPU + Family H2 cash defaults (2026-10-02) are **planning assumptions** until bake-off Accept.

## Documents

| Path | Owner / role | Status |
|------|----------------|--------|
| [architecture.md](./architecture.md) | Architect (draft) | Living overview |
| [adrs/](./adrs/) | Architect (draft); Leo confirms after bake-off | All ADRs **Proposed** |
| [cost-framing.md](./cost-framing.md) | Architect (draft) | CoS season-1 envelope (provisional, 2026-10-02) |
| [prd.md](./prd.md) | Tech Writer (draft) | Ready for Architect alignment |

## ADR index

See [adrs/README.md](./adrs/README.md) for status and one-line summaries.

## Product thesis (one sentence)

One-time heavy ingest (chunk → multimodal embed → vector + metadata index) of game video, then many cheap NL / player / play-type queries that assemble highlight reels **without** reprocessing raw video.

## Success gate (MVP experiment)

On **one real game**, a parent-style query returns useful clips in **&lt;5 s** from the index **without** re-sending the full video through an LLM.
