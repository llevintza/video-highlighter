# ADR-0005: Runtime and hosting

- Status: Proposed
- Date: 2026-10-01
- Deciders: Architect (draft); Leo to confirm after bake-off
- CoS provisional defaults: 2026-10-02 (planning assumptions; stack still **Proposed**, not Accepted)

## Context

Workloads have opposite shapes:

| Plane | Character | Needs |
|-------|-----------|-------|
| **Ingest** | Heavy, bursty, failure-tolerant, often GPU | Scale to zero; overnight OK |
| **Query + assemble** | Cheap, latency-sensitive, always reachable | Small CPU service; returns timestamps + signed clip URLs |

Co-locating both on one home box is fine for a private MVP **if** the *design* still separates planes (queue-driven ingest vs always-on API). Always-on cloud GPUs are an anti-pattern for this product.

**Stack is not locked.** Home GPU vs Modal/RunPod vs hyperscaler remains an experiment; options A–D below stay on the table until bake-off Accept. **CoS provisional default (2026-10-02):** plan **Modal-first** ingest until host `xoondev001` is online and `nvidia-smi` confirms a usable GPU. Prefer **home NVIDIA** if/when found (including on `xoondev001` or another home box). Keep Modal/RunPod as the burst path — never always-on cloud GPU.

## Decision drivers

- Protect the &lt;5 s query gate (API must not wait on GPU queues)
- Minimize idle GPU spend
- Privacy of youth athlete uploads
- Upload reality (gym Wi-Fi vs home)
- Room to grow to multi-user reliability without rewrite
- Pair with FFmpeg workers and optional Mux/Stream

## Options considered

### A. Home GPU / single workstation (MVP)

- Ingest overnight on local NVIDIA GPU (RTX 3060–4090 class often enough for SigLIP frame embedding; InternVideo2 may want more VRAM).
- Query API on same machine or tiny VPS.
- Pros: near-zero variable AI cost; privacy. Cons: uptime, upload bandwidth from gym, “spouse-factor” ops, no HA.

### B. Single VPS (Hetzner / DigitalOcean / Fly.io CPU) + occasional cloud GPU

- Always-on API + Postgres/Qdrant on VPS (~$5–40/mo).
- Burst embed jobs to **Modal**, **RunPod**, **Vast.ai**, or **AWS Batch + Spot GPU**.
- RunPod: consumer/pro GPUs by the second; spot/interruptible cheaper. Modal: Python-native serverless GPUs; great for sporadic index jobs; watch per-second minimums.
- Pros: pay per job; clear plane split. Cons: data egress to GPU vendor; secrets/ops for two environments.

### C. Hyperscaler batch + serverless API

- AWS: S3 + Batch/Spot GPU for ingest; Lambda/ECS/Fargate or App Runner for query; MediaConvert optional. Analogues on GCP/Azure.
- Pros: scale and IAM. Cons: bill complexity; idle GPU waste if not carefully scaled to zero.

### D. Fully managed video platforms (Mux / Stream) + thin app

- Hosting burden shifts to vendor; app is mostly auth + UX + vendor AI APIs.
- Conflicts with “own the embedding index forever” unless a side index still runs.

**MVP experiment starting point (not a lock):** start on **B with Modal-first** burst GPU until a home NVIDIA is confirmed. **A is preferred** when `xoondev001` (or another home box) shows a usable GPU via `nvidia-smi`. C–D remain future/alternate paths.

### Cost intuition (illustrative, ~2026-10-01)

| Path | Ingest 100-min game | Always-on API | Notes |
|---|---|---|---|
| Home GPU | ~$0 marginal | $0–15 VPS | Best early economics |
| Modal/RunPod T4/L4 ~0.5–1 h | ~$0.50–$2 | $10–30 VPS | Pay per job |
| TwelveLabs index instead of DIY GPU | ~$4.17 + infra | Thin API | Shifts cost to vendor AI (see ADR-0002) |
| Always-on cloud GPU | Wasteful | Avoid | Anti-pattern |

Season-1 planning band (CoS 2026-10-02): Family H2 year-one cash **~$50–150**; Modal ~$30/mo credits as primary ingest until a home GPU is confirmed. Club monthly bands are out of season-1 scope — [../cost-framing.md](../cost-framing.md).

## Recommendation (MVP experiment — not lock-in)

**Split planes early in design (even if co-located on one box at first):**

1. **Ingest worker** — queue-driven (e.g. SQS / Redis / Cloud Tasks / simple DB `jobs` table); scale to zero; GPU when needed; writes blobs + index.
2. **Query API** — small always-on service returning timestamps + signed clip URLs; assemble via FFmpeg job (or Stream/Mux clip APIs). Do **not** re-send full video through an LLM on the query path.

**MVP runtime starting point (CoS provisional default 2026-10-02 — not lock-in):** **Modal-first** GPU ingest until `xoondev001` is online and `nvidia-smi` confirms a usable home NVIDIA GPU. Prefer home NVIDIA when that check succeeds (including on `xoondev001` or another home box). Keep Modal/RunPod as the **burst** path; never always-on cloud GPU. Pair with one small VPS (or Fly/Railway) for API + DB. Promote to hyperscaler only when multi-user reliability demands it. Prefer FFmpeg assemble/export; defer Shotstack polish until branding needs it.

**Season-1 cash envelope (CoS provisional default 2026-10-02):** size the experiment to Family H2 year-one cash **~$50–150** (Family H2 / Tier 1). Do **not** plan season-1 against Club bands (~$200 soft monthly or ~$2k monthly); those remain *future* scenarios only. Detail: [../cost-framing.md](../cost-framing.md).

Frontend remains deferred; CLI/timestamp JSON can prove the thesis.

## Rationale

Cost profiles differ by an order of magnitude between overnight embed and parent queries after the game. Separating planes preserves the product thesis regardless of whether hardware is a basement RTX or a serverless GPU. Queue + job table also enables retries without blocking uploads.

## Consequences

**Positive**

- Query SLA insulated from ingest storms
- Scale-to-zero GPU aligns with “tens of games” early volume
- Clear promotion path to cloud without redesigning APIs

**Negative / risks**

- Home-lab SPOF and bandwidth limits for multi-parent sharing
- Cloud GPU paths need careful handling of private video egress
- Two-plane debugging (job stuck vs API healthy) needs minimal observability early
- Under-scoping assemble latency: long re-encodes should be async even when search is &lt;5 s

**Neutral**

- TwelveLabs H1 may shrink DIY GPU needs for understanding while still needing blob + query hosting

## Open questions

1. **Home GPU (CoS provisional default 2026-10-02).** Plan **Modal-first** until host `xoondev001` is online and `nvidia-smi` confirms a usable GPU. Prefer **home NVIDIA** if/when found (including on `xoondev001` or another home box). Remaining work is host-online verification, not a re-ask.
2. **Season-1 budget (CoS provisional default 2026-10-02).** Active envelope = Family H2 year-one cash **~$50–150** (Family H2 / Tier 1). Do **not** plan season-1 against Club bands (~$200 soft monthly or ~$2k monthly). Club figures remain future scenarios only. See [../cost-framing.md](../cost-framing.md).
3. Who uploads (parent phone from gym Wi-Fi vs home after game)?
4. Need multi-region or is US-East / home NAS enough?
5. Acceptable max ingest turnaround (same night vs next morning)?

## References

- Research brief §5 and closing table — `/workspace/video-highlighter-research-brief.md`
- Season-1 envelope: [../cost-framing.md](../cost-framing.md)
- Architecture overview: [../architecture.md](../architecture.md)
- [RunPod GPU pricing](https://www.runpod.io/gpu-instance/pricing)
- [RunPod vs Modal (Markaicode)](https://markaicode.com/vs/runpod-vs-modal/)
- Prices: snapshots ~2026-10-01
