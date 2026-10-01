# ADR-0001: Ingest and chunking

- Status: Proposed
- Date: 2026-10-01
- Deciders: Architect (draft); Leo to confirm after bake-off

## Context

Youth/HS/club basketball games arrive as long amateur recordings (~90–120 minutes, often ~2–8 GB). The product needs a **one-time ingest** that normalizes media, decides chunk boundaries, and retains precise time ranges so later export can cut clips **without** re-analyzing content.

Amateur sideline video is continuous: it rarely has broadcast-style hard cuts. Chunking strategy therefore drives index granularity, embed cost, and clip accuracy.

**Stack is not locked.** This ADR recommends an MVP **experiment starting point**, not a final media platform.

## Decision drivers

- Own absolute timestamps against the original (`game_id`, `start_ms`, `end_ms`)
- Preserve “query forever without reprocess”
- Control cost on multi-GB files (compute-bound, not per-minute vendor lock for basic cuts)
- Support later assemble via FFmpeg (or vendor clip APIs) from stored ranges
- Privacy: keep authoritative originals portable (family/youth athlete video)
- Success gate depends on index quality; chunking must be reliable on real gym footage

## Options considered

### A. Local/open pipeline: FFmpeg (+ optional PySceneDetect / adaptive detectors)

- Decode/probe with FFmpeg; chunk via fixed windows (e.g. 4–8 s), scene cuts, or hybrid (fixed + merge quiet/static).
- PySceneDetect AdaptiveDetector can reduce false cuts under camera motion (handheld/sideline).
- Export: store ranges; cut later. Frame-accurate cuts need re-encode; codec-copy snaps to keyframes (~±1–2 s typical).
- Cost: compute only (near-zero $ once hardware exists). Ingest offline/batch; assembly seconds–minutes.
- Ops: full control; you own codecs, proxies, failure recovery.
- Caveat: **pure scene-detect alone is a weak primary chunker** for continuous game recordings.

### B. Managed encode + host: Mux Video or Cloudflare Stream

- Upload → ABR encode → hosted playback. Mux Robots (beta) can prototype summaries/chapters/key moments with separate metering. Stream bills stored + delivered minutes; encoding included.
- Cost sketches (~2026-10-01 list): Mux 1080p basic storage ~$0.003/min/month, delivery ~$0.001/min after free tier; Cloudflare Stream ~$5/1,000 min stored, ~$1/1,000 min delivered.
- Fit: excellent **parent playback UX**; weaker as sole chunk+embed+metadata brain. Dual storage if originals must stay portable.

### C. Managed transcode only: AWS MediaConvert (or GCP Transcoder / Azure Media Services)

- Job-based transcoding from object storage; you keep files and build CDN delivery.
- Cost: MediaConvert normalized minutes — e.g. Basic HD H.264 ≤30 fps **2×** multiplier; first tier ~$0.0075/normalized minute → ~$0.015/real HD minute for one ladder rung before multi-rendition and S3/CloudFront (~2026-10-01).
- Fit: strong when already deep in AWS; overkill for early family MVP; egress often dominates later.

### D. Programmatic edit/render: Shotstack (and similar)

- JSON timeline → rendered highlight MP4 (titles, transitions, music).
- Cost: ~$0.20–$0.30/output minute (~2026-10-01).
- Fit: **assembler/polish after retrieval** — not a substitute for ingest/index. Expensive if every parent reel is fully re-rendered vs playlist-of-ranges / HLS stitch.

### E. Cloud video AI analyzers as ingest helpers

- Google Video Intelligence, Azure Video Indexer, Amazon Rekognition Video: shot/label/OCR/person APIs priced per analyzed minute.
- Useful as **metadata enrichment**, not primary media storage or chunk authority.

### Tradeoff snapshot

| Option | Cost at MVP | Ops | Long 1–2h games | Clip export later |
|---|---|---|---|---|
| FFmpeg (± scene detect) | Lowest $ | Medium | Excellent | Excellent (own ranges) |
| Mux / Stream | Low–medium | Lowest | Good (policy/limits apply) | Good if APIs expose clips |
| MediaConvert + S3 + CDN | Medium + egress risk | High | Excellent | Excellent |
| Shotstack | High per render | Low | N/A (assembler) | New files |
| Cloud label APIs | Medium–high / min | Low | Pay per minute | Ranges via annotations |

## Recommendation (MVP experiment — not lock-in)

**Primary ingest = FFmpeg-based worker** that produces:

1. A **normalized proxy** (e.g. 720p H.264), and  
2. A **chunk manifest** of fixed **~4–8 s** windows, with optional AdaptiveDetector merge for hard cuts.

Store absolute timestamps against the **original**. Use Mux or Cloudflare Stream **only if** parent streaming UX is a near-term priority and you accept dual storage (originals in blob + hosted playback). Defer MediaConvert until multi-tenant volume or ABR ladders justify AWS ops. Use Shotstack (or FFmpeg concat) **only at export**, after retrieval.

Prefer generating/caching clips on demand over materializing every reel.


## Provisional v0: period / half markers (manual)

Parent-style queries such as “second half” or “Q3” need a **period / half filter** on chunks. In v0 there is **no** automatic period derivation (no scoreboard OCR, silence detection, or game-clock ML). Do not pretend those exist yet.

**Provisional path (at ingest or shortly after):**

1. An operator records **manual period/half markers** as metadata ranges on the game — wall-clock or video timestamps for period starts/ends (e.g. Q1–Q4, halftime, OT). Store these as ranges on `games` (or a related `period_markers` table), not as inferred labels.
2. Chunks **inherit** `period` / `half` by **overlapping** those marker ranges (SQL or payload filter on metadata). Parent queries then filter without ML period detection.
3. **Out of v0 (optional later):** derive periods from scoreboard OCR, silence gaps, or visible game clock — evaluate only after the manual path works on real footage.

This keeps example queries like “#12 defense in the second half” honest: the filter works because markers were recorded, not because the system “knows” halves by itself.

## Rationale

Youth games are long continuous recordings—scene detectors designed for edited video under-segment. Owning time ranges preserves the product vision and avoids binding the index to a vendor’s clip IDs. Fixed temporal windows give predictable embed cardinality (~500–3,000 chunks/game depending on window size). Optional AdaptiveDetector helps only where real cuts exist.

## Consequences

**Positive**

- Portable, re-embeddable index keyed by time ranges
- Lowest variable $ for cutting/proxying once a worker box exists
- Clear handoff to assemble: FFmpeg `-ss`/`-to` (or re-encode for accuracy)

**Negative / risks**

- DIY reliability: packaging workers, monitoring, corrupt uploads, variable phone codecs
- Keyframe snap vs re-encode tradeoff on clip edges
- If day-1 streaming UX is mandatory, FFmpeg alone does not replace Mux/Stream player polish
- Period/half filters in v0 depend on operator-entered markers; missing markers → those filters return empty or unscoped results

**Neutral**

- Cloud label APIs remain optional enrichers on the same chunk timeline (see ADR-0002)

## Open questions

1. Typical source format/size (phone vs camcorder; 1080p60 vs 4K; average GB/game)?
2. Is “good enough phone streaming” required on day 1, or is downloadable MP4 / timestamp list enough for MVP?
3. Acceptable clip accuracy: keyframe-snapped (~±1–2 s) vs frame-accurate re-encode?
4. Will originals stay private (family-only) forever, or is sharing/public links planned?
5. Preferred fixed window length (4 vs 6 vs 8 s) after bake-off recall tests?

6. Who enters period/half markers in v0 (operator at ingest vs coach after upload), and is marker granularity Q1–Q4 + OT enough?

## References

- Research brief §1 — `/workspace/video-highlighter-research-brief.md`
- [PySceneDetect detectors](https://www.scenedetect.com/docs/latest/api/detectors.html)
- [Mux pricing](https://www.mux.com/pricing) / [Cloudflare Stream pricing](https://developers.cloudflare.com/stream/pricing/)
- [AWS MediaConvert pricing](https://aws.amazon.com/mediaconvert/pricing/)
- [Shotstack pricing](https://shotstack.io/pricing/)
- Prices: snapshots ~2026-10-01
