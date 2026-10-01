# ADR-0002: Moment detection and embeddings

- Status: Proposed
- Date: 2026-10-01
- Deciders: Architect (draft); Leo to confirm after bake-off

## Context

After chunking, the system must produce signals for (a) semantic/NL retrieval, (b) play-type filters, and (c) player filters—under **amateur angles, no broadcast graphics, small jerseys, motion blur**.

Commercial auto-highlight systems (Hudl autogen, Pixellot, WSC Sports) are tuned for broadcast or fixed-install multi-cam, often with humans in the loop. Sideline phone video will degrade jersey OCR, tracking, and “made basket” detection. Research datasets (BARD, E-BARD, PlayOn HS samples) are mostly NBA/broadcast-oriented—useful for prototypes, not proof that Leo’s gym footage will match.

**Stack is not locked.** This ADR defines a **bake-off**, not a single winning model.

**Privacy:** Defer **face biometrics for minors**. MVP player ID = roster + **manual tags**; jersey OCR best-effort only.

## Decision drivers

- Enable parent-style NL queries without re-sending full video through an LLM at query time
- Support play-type and player filters with human correction early
- One-time ingest cost shape; many cheap queries
- Measure quality on **real** HS/club footage before paying vendor gravity or DIY GPU forever
- Ethical/legal constraints for youth athletes
- Success gate: useful clips in &lt;5 s from index

## Options considered

### A. Managed multimodal video index (TwelveLabs Marengo / Search)

- Index once; semantic search; sports & broadcasting as an explicit vertical; Embed API for vectors (verify export/retention).
- Cost (~2026-10-01 Developer list): Search indexing ~**$2.50/hour** one-time + infra ~**$0.09/hour/month** indexed + search ~**$4/1k queries**. Free tier: up to 10 hours indexing, 90-day index access.
- Illustrative: 100-minute game ≈ $4.17 index + ~$0.15/mo infra; 50 games ≈ ~$208 one-time + ~$7.50/mo infra.
- Tradeoffs: fastest path to “NL over video”; vendor gravity; amateur quality **unproven until tested**.

### B. Cloud video annotation APIs (Google Video Intelligence, Azure Video Indexer, Amazon Rekognition Video)

- Structured labels, shots, OCR, person/face APIs priced per minute.
- GCP example (~2026-10-01): Label $0.10/min, Shot $0.05/min (free with Label), OCR $0.15/min, Object tracking $0.15/min, Person/Face $0.10/min after free tiers → stacking label+OCR+person ≈ **$0.35+/min** → ~$35+ per 100-min game.
- Rekognition similar stacking risk; Azure Video Indexer bundles presets per input minute.
- Tradeoffs: good **enrichment**; **not** basketball-play taxonomy out of the box (“person” ≠ “steal by #12”).

### C. Generative multimodal LLMs on video (Gemini video understanding; OpenAI via frames)

- Gemini: native video via Files API; ~1 FPS default sampling; can return timestamped descriptions; token-priced.
- OpenAI: generally frame/image pipelines for long games—more DIY.
- Tradeoffs: excellent for **offline captioning / play-type labeling of chunks**. Expensive/slow if every parent query re-sends full video — fits architecture **only at ingest**.

### D. Self-hosted / open multimodal embeddings (SigLIP2 / OpenCLIP / InternVideo2 / VideoCLIP-class)

- SigLIP2 / OpenCLIP: strong image–text; typically sample frames and pool for video.
- InternVideo2: stronger temporal/action modeling; heavier GPU needs ([arXiv:2403.15377](https://arxiv.org/html/2403.15377v4)).
- Tradeoffs: you own vectors forever; one-time GPU cost aligns with “ingest once.” Quality gap vs TwelveLabs on sports semantics may be large until fine-tuned; youth basketball fine-tune data is scarce.

### Player identification options

| Method | Pros | Cons for youth games |
|---|---|---|
| Jersey OCR (track → crop → OCR/VLM) | Matches coach language (“#12”) | Small digits, blur, occlusions, duplicate numbers |
| Team color + number | Reduces collisions | Similar uniforms; lighting |
| Face recognition | Persistent ID | Consent/COPPA/privacy for minors; ethics; faces change |
| MOT + ReID | Continuity within a possession | Pans lose IDs; calibration |
| **Manual / roster tags (MVP)** | Reliable for “my kid” | Labor; needs a thin coach/parent UI |

### Approaches compared

| Approach | NL quality | Play-type | Player ID | Cost shape | Ops |
|---|---|---|---|---|---|
| TwelveLabs index | High (vendor) | Via search / classify | Weak alone | $/hour indexed + monthly infra | Low |
| Cloud label APIs | Low–medium | Coarse labels | OCR/person helpers | $/min × features | Low |
| Gemini/LLM ingest captions | High if prompted well | Can emit taxonomy | Can attempt jersey read | Tokens × chunks | Medium |
| Self-host SigLIP/InternVideo2 | Medium → high with tuning | Needs head or captions | Separate CV stack | GPU batch | Medium–high |
| Hybrid (embed + LLM labels + OCR + human) | Best | Best | Best | Balanced | Medium |

## Recommendation (MVP experiment — not lock-in)

**Hybrid bake-off on 2–3 real games:**

1. **Path H1 (speed):** TwelveLabs Search index on full games → measure recall for prompts like “steals”, “three-pointers”, “#12 layup” on *your* footage; track monthly infra vs quality.
2. **Path H2 (owned index):** FFmpeg chunks → **SigLIP2** (or **InternVideo2** if GPU allows) embeddings into your vector DB + **batch Gemini Flash** (or similar) to propose play-type tags + jersey guesses into metadata → human correct once.
3. Keep cloud Video Intelligence / Rekognition / Azure Video Indexer as **optional OCR/shot helpers**, not the primary brain—watch stacked $/min.

**Player ID for MVP:** Roster CSV (number → name) + **human confirmation** of key clips; jersey OCR as best-effort with confidence scores. **Defer face recognition for minors** until legal/product review.

Do **not** assume CLIP alone solves basketball semantics; do **not** assume NBA-trained action models transfer to phone video without evaluation. Do **not** call full-video LLMs at query time (breaks the &lt;5 s / no-reprocess gate).

## Rationale

Quality on amateur HS video is an empirical question. H1 minimizes time-to-signal; H2 maximizes ownership and aligns with portable vectors. Manual tags acknowledge that “my daughter only” is the highest-value filter and is unreliable from pure CV at MVP. Separating ingest-time LLM captions from query-time ANN is what makes the success gate achievable.

## Consequences

**Positive**

- Explicit bake-off prevents premature vendor or DIY lock-in
- Human-in-the-loop matches how youth sports highlight products actually ship quality
- Query path stays cheap (text embed + ANN + metadata filters)

**Negative / risks**

- H1: monthly infra while indexed; embed export policies need verification
- H2: GPU ops, labeling UX, possible weaker sports semantics until tuned
- OCR false confidence can mislead parents if shown without scores
- Dual-path experiments cost calendar time and some $

**Neutral**

- Taxonomy of play types remains a product decision (coach-defined vs fixed list)

## Open questions

1. Can Leo share 1–2 representative game files under private access for bake-offs?
2. Minimum viable player filter: “my daughter only” via manual tag vs automatic jersey?
3. Required play taxonomy (shot/make/miss, rebound, steal, assist, foul, hustle…)? Coach-defined?
4. Hard privacy constraints confirmed (no face biometrics for minors; region data residency)?
5. Acceptable false-positive rate for parent reels (extra dull clips vs missing a big play)?
6. Preference: TwelveLabs lock-in comfort vs DIY GPU time / budget?

## References

- Research brief §2 — `/workspace/video-highlighter-research-brief.md`
- [TwelveLabs pricing](https://www.twelvelabs.io/pricing) / [sports & broadcasting](https://www.twelvelabs.io/solutions/sports-and-broadcasting)
- [Google Video Intelligence pricing](https://cloud.google.com/video-intelligence/pricing)
- [Amazon Rekognition pricing](https://aws.amazon.com/rekognition/pricing/)
- [Azure Video Indexer pricing](https://azure.microsoft.com/en-us/pricing/details/video-indexer/)
- [Gemini video understanding](https://ai.google.dev/gemini-api/docs/video-understanding)
- [SigLIP2 docs](https://huggingface.co/docs/transformers/model_doc/siglip2) / [InternVideo2](https://arxiv.org/html/2403.15377v4)
- [BARD](https://github.com/GabrieleGiudic/BARD) / [E-BARD](https://github.com/GabrieleGiudic/E-BARD/) / [PlayOn samples](https://github.com/playon/basketball-analysis-samples)
- Prices: snapshots ~2026-10-01
