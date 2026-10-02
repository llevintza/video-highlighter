# Season-1 cost framing (planning assumptions)

- **Status:** CoS provisional defaults (2026-10-02) — not a bake-off Accept
- **Owner:** Architect (draft)
- **Does not lock the stack.** Runtime options A–D remain open in [ADR-0005](./adrs/0005-runtime-and-hosting.md).

This is a short envelope note, not a rewrite of the research brief’s cost tables.

## Active season-1 envelope

**Family H2 / Tier 1 year-one cash: ~$50–150.**

Size ingest, storage, and always-on query spend to this band. It is the **active** season-1 planning assumption.

**Out of season-1 scope:** Club bands (~$200 soft monthly, or ~$2k monthly) are **future scenarios only**. Do not plan season-1 against them.

## GPU / ingest spend (until a home GPU is verified)

| Assumption | Planning default |
|---|---|
| Primary ingest runtime | **Modal-first** — treat Modal **~$30/mo credits** as the primary ingest budget until host `xoondev001` is online and `nvidia-smi` confirms a usable GPU |
| Preferred when found | **Home NVIDIA** (on `xoondev001` or another home box) — near-zero marginal ingest $ |
| Burst / overflow | Modal or RunPod **per job** — never always-on cloud GPU |

When home NVIDIA is confirmed, Modal/RunPod stay the burst path, not an always-on default.

## What this does *not* decide

- H1 (TwelveLabs) vs H2 (owned embeddings) bake-off — [ADR-0002](./adrs/0002-moment-detection-and-embeddings.md)
- Blob vendor (R2 / B2 / NAS) — [ADR-0004](./adrs/0004-blob-storage.md)
- Accepted runtime — [ADR-0005](./adrs/0005-runtime-and-hosting.md) stays **Proposed**

## References

- [ADR-0005 — Runtime and hosting](./adrs/0005-runtime-and-hosting.md)
- [Architecture overview](./architecture.md)
