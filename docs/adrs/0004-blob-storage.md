# ADR-0004: Blob storage

- Status: Proposed
- Date: 2026-10-01
- Deciders: Architect (draft); Leo to confirm after bake-off

## Context

The product stores multi-GB **originals**, smaller **proxies**, and optionally **exported reels**. Parents and coaches may stream or download short highlights (1–10 min) to phones. Egress—not storage—often dominates video bills.

Privacy: family/youth athlete video. Keep **authoritative originals portable** (re-embed, leave a vendor player, comply with school/club rules). Prefer storing original + proxy and **generating/caching clips on demand** over materializing every combinatorial reel.

**Stack is not locked.** NAS vs R2 vs B2 (vs S3-class) is an experiment starting point tied to who watches remotely.

## Decision drivers

- Protect originals; checksums; re-ingest capability
- Avoid surprise egress when parents stream after games
- S3-compatible APIs where possible for tooling reuse
- Acceptable private MVP (LAN/VPN) vs shared remote parents
- Compliance: where minor athlete video may live
- Pair cleanly with FFmpeg assemble and optional Mux/Stream playback (dual storage if used)

## Options considered

| Option | Storage (approx list ~2026-10-01) | Egress to internet | Notes |
|---|---|---|---|
| **Cloudflare R2** | $0.015 / GB-month (Standard); 10 GB free | **Free** egress | S3-compatible; Class A/B ops charges |
| **Backblaze B2** | $6.95 / TB-month (~$0.00695 / GB) | Free up to **3×** monthly storage; then $0.01/GB; free via CDN partners | Strong archive + delivery economics |
| **Wasabi** | ~$7.99 / TB-month | “Free” with **fair-use** (egress ≲ stored volume); overage risk | Cheap storage; read policy carefully before hot streaming |
| **AWS S3 Standard** | ~$0.023 / GB-month (US East, first tier) | ~$0.09 / GB typical internet egress | Expensive egress for parent streaming |
| **GCS / Azure Blob** | ~$0.020 / GB-month class dependent | Egress metered (similar pain) | Fine if already on that cloud |
| **Local NAS / home disk** | CapEx + electricity | LAN free; remote needs VPN/Tailscale | Excellent private MVP; weak multi-user CDN |

**Worked storage sketch (from brief):** 50 games × 5 GB = 250 GB ≈ **$3.75/mo on R2** or ~**$1.74/mo on B2** (storage only). If parents stream **500 GB/month** of highlights, S3-class egress could be ~**$45+**, while R2 is **$0** (plus request charges); B2 may still be free if under 3× storage.

**Clips strategy:** Prefer **original + proxy** + on-demand (or cached) highlights vs pre-materializing every reel.

## Recommendation (MVP experiment — not lock-in)

- **MVP private (family-only):** Local **NAS** or external SSD + checksums for originals; optional offsite backup.
- **MVP shared / parents remote:** **Cloudflare R2** (or **B2 + Cloudflare/Fastly**) for free/cheap egress; keep S3 only if the rest of the stack is already AWS-centric and you front with CloudFront carefully.
- Avoid **Wasabi** as primary **hot streaming** origin until fair-use fit is modeled.
- If using Mux/Stream for playback polish, still keep **authoritative originals** in R2/B2/NAS for re-embed and portability.

Assemble via FFmpeg from proxy when quality allows; fall back to original for higher-accuracy cuts. Cache popular export blobs with TTL keyed by query fingerprint (see architecture data sketch).

## Rationale

Egress sensitivity matches bursty post-game viewing. Portability of originals is non-negotiable for youth privacy and for swapping understanding stacks (H1/H2). On-demand clip generation avoids combinatorial storage growth as roster × play-type × game queries explode.

## Consequences

**Positive**

- Predictable early storage bills; R2/B2 favor delivery economics
- NAS path maximizes privacy and near-zero $ for family-only MVP
- Clear “source of truth” for reprocessing

**Negative / risks**

- Dual storage if Mux/Stream is added (sync/deletion discipline)
- NAS remote access UX (Tailscale/VPN) may frustrate multi-parent sharing
- Wasabi fair-use surprises if misused as hot CDN
- S3 without careful CDN/fronting can blow the season budget on egress alone

**Neutral**

- Retention policy (forever vs N seasons) drives capacity more than vendor choice at tens–hundreds of games

## Open questions

1. Expected concurrent remote viewers (family only vs whole team)?
2. Retention: keep every game forever, or rolling N seasons?
3. Geographic audience (US-only) affecting region choice?
4. Compliance: any school/club rules on where minor athlete video may live?
5. Is downloadable MP4 enough for MVP, or must streaming work from day 1?

## References

- Research brief §4 — `/workspace/video-highlighter-research-brief.md`
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Wasabi pricing FAQ](https://wasabi.com/pricing/faq)
- [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)
- Prices: snapshots ~2026-10-01
