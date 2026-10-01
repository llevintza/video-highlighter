# Product Requirements Document — Video Highlight Creator

**Status:** Draft (for Architect / Tech Coordinator / Code Reviewer)  
**Repo:** [llevintza/video-highlighter](https://github.com/llevintza/video-highlighter)  
**Audience:** Product + engineering (Architect ADRs + architecture overview live alongside this PRD)  
**Source brief:** `/workspace/video-highlighter-research-brief.md` (prepared 2026-10-01 EDT) — cite for options, cost sketches, and open questions  
**Companion docs:** [architecture.md](./architecture.md), [adrs/](./adrs/) (Architect-owned; stack **not** locked)

> **How to read stack language in this PRD.** Infrastructure choices are framed as **options → MVP experiment recommendation → open questions**. Nothing here fiat-locks FFmpeg, TwelveLabs, pgvector, R2, or any other vendor. The research brief’s coherent MVP table is a **bake-off starting point** only. Architecture Decision Records capture provisional decisions; Leo confirms after experiments on real footage.

---

## 1. Problem

Parents, coaches, and players already film youth / high-school / club basketball games (often 1–2 hours of sideline or phone video). Turning those recordings into **personalized highlight reels**—by player, play type, game segment, or natural language—is slow, repetitive, and usually means scrubbing the full video again for every request.

Commercial auto-highlight tools are tuned for broadcast or fixed multi-cam installs. Amateur angles, motion blur, small jerseys, and no broadcast graphics mean “NBA-quality” auto labels will not transfer reliably. Families still want something better than manual editing for every reel.

**Product thesis (from the research brief):** ingest a game **once** with a heavy multimodal pipeline (chunk → embed → vector + metadata index), then answer **many cheap queries** that assemble highlight reels **without** reprocessing the raw video through an LLM or re-analyzing the full file.

---

## 2. Users & stakeholders

| Persona | Need |
|---------|------|
| **Parent** | “Show me my kid’s defensive plays from the second half” — short, shareable clips without scrubbing 2 hours. |
| **Coach** | Filter by play type / period / roster for film review and teaching clips. |
| **Player** | Personalized reels for recruiting, self-review, or sharing with family. |
| **Uploader / family operator** (often Leo for v0) | Get a game from phone/camcorder into the system once; tolerate overnight ingest; keep footage private. |

Primary v0 setting: **family / small club**, not multi-tenant SaaS. Multi-family or whole-club tenancy is an open timeline question (see §8).

---

## 3. Goals

1. **Ingest once, query many** — durable index of time-ranged chunks + metadata so queries never require full-video re-LLM.
2. **Parent-style retrieval** — NL and structured filters (player, play type, game, time range) return useful clips.
3. **Honest amateur-video bar** — hybrid human-in-the-loop early (roster tags, coach/parent corrections); not fully autonomous broadcast quality.
4. **Privacy-first for minors** — no face biometrics in v0; portable ownership of originals and index where feasible.
5. **Cost-shaped for season 1** — bursty GPU/ingest cost; cheap always-on query path; egress-aware delivery when parents stream remotely.

---

## 4. Success metrics

### MVP success gate (must-pass — latency + quality)

The gate has **two** parts. Latency alone is not enough.

**A. Latency / no-reprocess (query path)**  
On a warm index for **one real gold game**, parent-style queries return ranked clip ranges in **&lt;5 seconds** (p50) **without** re-reading or re-sending the **full** video through an LLM. Latency budget covers retrieve + optional short assemble, not overnight ingest.

**B. Quality — minimal eval protocol (same gold game)**  
Before calling H1 or H2 a bake-off winner, run this protocol:

| Step | What |
|------|------|
| Gold game | One representative youth/HS/club game under private access (same file for both paths). |
| Key-play labels | Before scoring: Leo and/or a coach pre-mark a set of **key plays** (time ranges + play-type / player tags) on that game — the recall ground truth. |
| Query set | **N ≥ 10** labeled parent-style queries (written before scoring), covering NL + filter style (player, play type, period). Example: “show #12 defense in the second half”. |
| Top-K | System returns **K = 5** clip ranges per query. |
| Judges | **Leo and/or coach** (named on the eval sheet). Judges mark each returned clip as relevant / not for that query. |
| Precision@5 | Fraction of the K=5 clips judged relevant, averaged over the N queries. |
| Key-play recall | Fraction of pre-marked key plays that appear in at least one top-K result for a query that should retrieve them (document mapping rules on the eval sheet). |
| Pass bar (provisional) | A path **passes** only if it meets **all** of: (1) latency gate A, (2) **Precision@5 ≥ 0.4**, (3) **key-play recall ≥ 0.5**, on the same gold game and query set. Bars are starting thresholds — Leo may raise them after the first eval. |
| H1 vs H2 | Run the **same** gold game, query set, and judges on both paths. Prefer the path that passes with higher Precision@5 then higher recall; if only one passes, that path is the provisional MVP experiment choice. If neither passes, document failures and iterate (tags, chunking, prompts) — do **not** lock a vendor. |

**H1 privacy precondition (must match ADR-0002):** do **not** run the H1 (managed vendor index, e.g. TwelveLabs) bake-off on team footage until Leo confirms consent / club rules for uploading video that includes **other minors** (teammates and opponents visible in the recording) to that vendor — see §6. H2 may proceed on an owned / on-premises stack under family-operator controls without that vendor-upload step. Face biometrics remain out of scope for minors on both paths.

### Supporting signals (v0)

| Signal | Intent |
|--------|--------|
| Query latency | p50 retrieve &lt;5 s from warm index (gate A). |
| Precision@5 / key-play recall | Gate B on the gold-game eval sheet. |
| Reprocess avoidance | Same game answers N follow-up queries with **no** second full-video AI pass. |
| Player filter usability | “My kid” works via roster + manual tags even if jersey OCR is weak. |
| Privacy | No face-biometric pipeline for minors in v0; H1 vendor upload gated on teammate/minor consent (§6 / ADR-0002). |

Quantitative season-1 cost/usage targets depend on Leo’s budget ceiling (open question).

---

## 5. MVP scope

### In scope (v0 experiment)

- Accept long amateur basketball recordings (youth / HS / club quality).
- **One-time ingest** producing: normalized proxy, chunk time ranges against the original, multimodal index + metadata suitable for filters and NL.
- **Query API** (or CLI) returning timestamps and paths/URLs for clips; assemble short reels on demand (cut/concat or playlist of ranges).
- **Player identity:** roster CSV (number → name) + **manual confirmation/tags** for key clips; jersey OCR as best-effort metadata with confidence scores.
- **Bake-off**, not a single locked understanding stack:
  - **H1:** managed multimodal video index (TwelveLabs Search) on full games.
  - **H2:** owned embeddings (e.g. SigLIP2 / InternVideo2 if GPU allows) + batch multimodal captions/tags (e.g. Gemini) into our vector/metadata store.
- Durable storage of originals + proxies; index versioning awareness so model changes can re-embed later.
- Documented provisional architecture (Architect ADRs) aligned to the brief’s experiment stack.

### Explicit non-goals (v0)

- Locked production stack or vendor commitment before bake-off on real HS/youth footage.
- Broadcast-quality autonomous play classification without human correction.
- **Face recognition / face biometrics for minors.**
- Full parent-facing polished UI / Next.js app (CLI or timestamp JSON may prove the thesis; frontend deferred).
- Pre-materializing every combinatorial highlight reel.
- Multi-tenant club SaaS, billing, or school LMS integrations.
- Shotstack (or similar) branded timeline polish unless branding becomes a near-term need.
- Always-on cloud GPU for idle cost.

### Framing for engineering (not PRD lock-in)

The research brief’s coherent MVP **experiment** stack (FFmpeg fixed chunks; TwelveLabs vs owned embeddings bake-off; Postgres + pgvector or Qdrant if needed; NAS and/or R2/B2; queue + GPU worker + small API; on-demand FFmpeg assemble) is the **default recommendation to try**. See Architect ADRs for options, provisional decisions, consequences, and follow-ups. This PRD requires only that the product meet the success gate and privacy constraints above.

---

## 6. Privacy, safety, and compliance (v0)

| Constraint | Requirement |
|------------|-------------|
| Minors | Treat athletes as minors by default for youth/HS footage. Team recordings typically include **other minors** (teammates, opponents, bystanders), not only the operator’s child. |
| Face biometrics | **Out of scope for v0** — do not ship face recognition / face match for player ID. |
| Player ID | Roster + manual tags first; jersey OCR best-effort only. |
| Teammate / club consent | Even without face biometrics, **processing and (for H1) uploading** full-game video of other minors may require club/school consent or parental agreement. **Open question for Leo** — see §8. Do not start H1 vendor upload until answered. |
| Consent / sharing | Prefer private family (or small trusted) access until Leo decides public/share links. |
| Data residency / school rules | Open question — may restrict where originals live and which vendors may receive copies. |
| Vendor AI (H1) | **Precondition (ADR-0002):** document what leaves the premises, retention, and subprocessors; keep authoritative originals under operator control. H1 bake-off is blocked until Leo confirms teammate/minor consent / club rules for vendor upload. |
| Owned stack (H2) | Prefer keeping bytes and embeddings under operator control for the bake-off when consent for vendor upload is unclear. |

---

## 7. Experience sketch (MVP, UI deferred)

1. **Upload** a game file (home after the game is enough for v0; gym Wi-Fi upload is an open ops question).
2. **Ingest job** runs offline/batch (minutes to hours) → proxy + chunks + index/metadata.
3. **Human tags** (optional but expected early): confirm “my kid” / key plays as needed.
4. **Query** via CLI or thin API: NL and/or filters → ranked time ranges (must meet §4 gate A latency and gate B quality on the gold-game eval).
5. **Assemble** short reel (1–10 min target class) → downloadable MP4 and/or signed URL; streaming polish optional later.

Delivery UX (in-browser stream vs download MP4) is an open question for Leo; either is acceptable if the §4 success gate (latency **and** quality) holds.

---

## 8. Open questions for Leo

Prioritized from the research brief’s closing list. Answers unstick bake-offs and ADR confirmation.

1. **Footage access** — Can Architect / implementers get 1–2 representative full games (length, resolution, angle) under private access for H1 vs H2 bake-offs?
2. **Budget ceiling** — Soft monthly cap for AI + hosting in season 1 (~$20 vs ~$200 vs ~$2,000)?
3. **Player ID bar** — Is manual roster tagging acceptable for v0 “my daughter only” filters?
4. **Delivery UX** — Stream-in-browser required on day 1, or is downloadable MP4 / timestamp list enough?
5. **Privacy / face biometrics** — Confirm face biometrics remain off-limits for minors in v0; any school/club rules or residency constraints on where minor athlete video may live?
5b. **Teammate / other-minors consent** — Team footage includes other minors. What club/school/parental consent (if any) is required to (a) process teammates’ video locally, and (b) upload full games to a managed AI vendor (H1 / TwelveLabs) even when face biometrics are unused? Who can approve that for bake-off?
5c. **H1 go/no-go** — Explicit yes/no to run H1 vendor indexing on a gold game that shows teammates/opponents, after 5b is answered.
6. **Taxonomy** — Who defines play types (shot/make/miss, rebound, steal, …)? Any existing Hudl/stat tags to reuse?
7. **Multi-tenant timeline** — Family-only this season, or whole club?
8. **Model ownership** — Comfort with TwelveLabs lock-in/infra fees vs DIY GPU time for an owned index?
9. **Source media** — Typical format/size (phone vs camcorder; 1080p60 vs 4K; average GB/game)?
10. **Clip accuracy** — Keyframe-snapped (~±1–2 s) OK, or frame-accurate re-encode required?
11. **False-positive preference** — Extra dull clips vs missing a big play?
12. **Local GPU** — Is a home NVIDIA GPU available for overnight ingest jobs?

---

## 9. Out-of-band dependencies

| Dependency | Owner | Notes |
|------------|-------|-------|
| Research brief | Tech Research (done) | Options + cost sketches; re-verify vendor prices before budgeting. |
| ADRs + architecture overview | Architect | Ingest, understanding bake-off, storage, runtime; ingest vs query planes. |
| Bake-off on real games | Architect + Implementer (after footage) | H1 TwelveLabs vs H2 owned embeddings + captions. |
| Repo PR / review | Architect lands set → Tech Coordinator → Code Reviewer | Prefer PR into `llevintza/video-highlighter`; else keep `/workspace/video-highlighter-docs/`. |

---

## 10. Document history

| Date | Author | Change |
|------|--------|--------|
| 2026-10-01 | Tech Writer | Initial draft from research brief + Architect guidance; stack framed as experiment only. |
| 2026-10-01 | Tech Writer | Code Reviewer patch: §4 eval protocol (Precision@5, key-play recall, judges, H1/H2 pass bar); §6 teammate-minors consent + H1 vendor precondition aligned with ADR-0002. |

*End of PRD draft. Stack decisions remain unlocked pending bake-offs and Leo’s answers above.*
