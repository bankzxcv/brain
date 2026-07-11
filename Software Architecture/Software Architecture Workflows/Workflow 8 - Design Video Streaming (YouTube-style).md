---
title: "Workflow 8 - Design Video Streaming (YouTube-style)"
date: 2026-07-11
tags:
  - software-architecture
  - system-design
  - video-streaming
  - cdn
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 8 — Famous Case: Video Streaming (YouTube-style)

> [!abstract] Project
> Design video upload, processing, and playback at scale. This case teaches three things the previous workflows didn't: **pipeline architecture in production** ([[Software Architecture Study#7.4 Pipeline (Pipes & Filters) Architecture|Study §7.4]] at massive scale), **the CDN as the real serving tier**, and **asymmetry** — upload and playback are so different they're effectively two separate products sharing a catalog.

**Architecture (target):**

```
 UPLOAD (write path — heavy, async, rare)
 creator ──► upload svc ──► raw blob storage
                               │ event: video.uploaded
                               ▼
        ┌── TRANSCODING PIPELINE (queue-driven workers) ──┐
        │ split→ transcode (per resolution ladder:        │
        │ 240p…4K, per codec) → thumbnails → captions →   │
        │ package (HLS/DASH segments) → QC → publish      │
        └───────────────┬─────────────────────────────────┘
                        ▼
              processed segments → origin storage ──► CDN (edge caches, ~everything)
                        │
                 metadata service → catalog DB + search index

 PLAYBACK (read path — light per-request, colossal in volume)
 viewer ──► player fetches manifest ──► CDN edge: segment.ts, segment.ts, ...
            (adaptive bitrate: player picks quality per network)
            API only for: metadata, auth, recommendations, view counts
```

---

## Step 1 — Requirements & the brutal numbers

**Do:** scope (upload, transcode, playback, metadata/search; skip live-streaming). Estimate: 2M uploads/day (avg 10 min), 1B watch-hours/day. Compute storage/day and egress bandwidth, then state the economic fact that dictates the whole architecture.

> [!success]- Solution
> Storage: 2M × 10 min × ~5MB/min raw ≈ 100TB/day raw; ×2–3 after all renditions ≈ **~300TB/day, ~100PB/year**. Egress: 1B watch-hours/day ≈ 41M concurrent-hours; at ~3 Mbps average ≈ **tens of Tbps** sustained.
> **The economic fact:** at Tbps scale, *bandwidth is the product cost*, and serving from origin servers is impossible — physics (speed of light to distant users = buffering) and economics (egress fees) both mandate a **CDN**: >95% of bytes must come from edge caches near viewers. Characteristics: playback smoothness (rebuffer rate — the metric users churn on) > startup time (< 1–2s) > availability > upload latency (creators tolerate minutes) ≫ consistency (a video appearing globally 2 min late is invisible). The asymmetry is total: optimize the read path with hardware, the write path with patience.

## Step 2 — The upload & transcoding pipeline

**Do:** design the transcoding pipeline as pipes-and-filters. Why must it be async and queue-driven? How do you make a 4K, 3-hour upload not block the 30-second phone clip?

> [!success]- Solution
> Filters: `validate → split-into-chunks → transcode(chunk × ladder-of-renditions) → merge → thumbnail → caption-extract → package(HLS segments + manifests) → QC → publish-event`. Each filter = stateless workers on a queue ([[Software Architecture Study#7.4 Pipeline (Pipes & Filters) Architecture|pipeline]] + [[Software Architecture Study#8.0 Synchronous & Asynchronous Communication|async]]): transcoding is minutes of CPU/GPU — anything sync would time out; queues absorb the daily upload tide; failed chunks retry individually.
> **Chunking is the key trick:** split video into ~10s chunks, transcode chunks *in parallel* across hundreds of workers → the 3-hour movie takes barely longer than the phone clip (wall-clock = slowest chunk, not total length). Separate queues per priority/size class prevent head-of-line blocking. Spot/preemptible instances cut compute cost ~70% — chunks are idempotent and retryable, the perfect preemptible workload ([[06 - Trade-off Analysis Drills#Problem 5 Cost model duel|cost thinking]]).
> Creator UX: "processing… we'll notify you" — eventual consistency presented honestly, like [[Workflow 3 - Microservices and Event-Driven on Cloud#Step 5 — The checkout saga across services|W3's checkout]] but with minutes of budget instead of seconds.

## Step 3 — Adaptive bitrate: why the player is part of the architecture

**Do:** explain HLS/DASH segmented streaming and adaptive bitrate (ABR). Why do segments + a dumb CDN beat a smart streaming server? What did this move *off* the server?

> [!success]- Solution
> Each video is packaged as a ladder of renditions (240p→4K), each cut into ~4–10s segments — plain HTTP files — plus a manifest listing them. The **player** measures its own throughput and requests the next segment at whatever quality fits: network dips → it downshifts; recovers → upshifts. That's ABR — smooth playback on wildly varying networks with zero server intelligence.
> Segments-over-HTTP means the "streaming server" is… a file server → **every CDN on earth can serve it**, caching works natively (segments are immutable — cache forever, [[#Step 4 — CDN strategy & the long tail|Step 4]]), and scale-out is trivial. The smart-server alternative (RTMP-era stateful streaming) required per-viewer server sessions — unscalable and CDN-hostile.
> Architectural lesson: **pushing intelligence to the client turned a hard stateful-server problem into a solved static-content problem.** Deciding *where* a responsibility lives is often worth more than optimizing it — compare [[Workflow 7 - Design a Chat System (WhatsApp-style)#Step 3 — The message flow A sends to B|chat's pull-as-truth]] inversion.

## Step 4 — CDN strategy & the long tail

**Do:** 10% of videos get 90% of views. Design the caching tiers, and handle both ends: a video going viral in one hour, and the 4-year-old video with 3 views/month.

> [!success]- Solution
> Tiers: **edge PoPs** (hundreds, near users, cache hot segments) → **regional mid-tier caches** (fewer, bigger — shield the origin, absorb edge misses) → **origin** (blob storage, everything). Immutable segments = no invalidation problem, pure LRU-by-popularity works.
> **Viral video:** first viewers per PoP miss → *request coalescing* (one origin fetch per segment per PoP, others wait on it) prevents origin stampede; within minutes the tail of PoPs is warm and origin load returns to ~zero. Optionally pre-push predicted-hot content (new upload from 50M-subscriber channel → warm top renditions proactively).
> **Long tail:** never cached, served from origin via mid-tier, slightly slower start — *correct*, not a flaw: caching 100PB everywhere is economically insane; the 3-view video doesn't merit edge storage. Storage tiering mirrors it: hot (SSD-backed) / warm / cold-archive renditions, re-transcode-on-demand for ancient rarely-watched quality levels — trading compute-on-rare-access for petabytes of standing storage.

## Step 5 — Metadata, search, and view counts

**Do:** the "small data" side: catalog metadata, search, and the deceptively hard one — view counts at 1B/day. Design each; why do exact real-time counters break, and what's the standard answer?

> [!success]- Solution
> **Metadata** (title, channel, stats): sharded SQL/document store + heavy caching — reads dominate, sizes are tiny; this is a normal [[Workflow 2 - Modular Monolith on a Cloud VM|W2-grade]] service hiding inside the giant.
> **Search:** index built asynchronously from `video.published` events ([[Software Architecture Study#9.3 CQRS — Command Query Responsibility Segregation|CQRS projection]], same shape as [[Workflow 6 - Design a News Feed (Twitter-style)#Step 6 — Ranking, ads, and the second read model|the feed's read models]]).
> **View counts:** 1B increments/day against single rows = hot-row death (every architecture meets its [[04 - Distributed Styles Drills#Problem 8 Space-based or not?|contention problem]] eventually). Standard answer: **count approximately in stream, reconcile in batch** — view events → Kafka → streaming aggregation (per-video counters flushed every few seconds) → display counter; exact ledger computed in batch for creator payments (where money = [[Workflow 4 - CQRS and Event Sourcing Service|W4-grade]] accuracy). Two accuracy classes for one number, priced separately — the per-data-class guarantees lesson from [[Workflow 7 - Design a Chat System (WhatsApp-style)#Step 6 — Presence, receipts, and typing indicators|chat's presence]] again.

## Step 6 — Failure modes

**Do:** predict and mitigate: (a) transcoding backlog after a holiday upload spike, (b) a mid-tier cache region fails, (c) origin blob store has elevated latency, (d) a bad transcode ships (green-tinted 1080p) for a video already viral.

> [!success]- Solution
> - (a) Queue depth grows; playback wholly unaffected (paths are independent — the asymmetry paying off). Autoscale workers; priority lanes keep short/priority videos flowing; creators see longer "processing" — the cheapest possible degradation.
> - (b) Edges re-route to sibling region mid-tier (higher latency, more origin load); rebuffer rate ticks up regionally; nothing is down. Capacity headroom per region is the mitigation you bought in advance.
> - (c) Cache hit ratio insulates viewers (>95% never touch origin); misses get slow starts. Alert, fail over storage region if sustained.
> - (d) Immutable segments can't be edited — publish *new* segment versions under new URLs + updated manifest (players pick up on next manifest refresh); CDN needs no purge (old URLs age out). Immutability turns "global emergency purge" into "publish a correction" — the same edit-by-append philosophy as [[Workflow 4 - CQRS and Event Sourcing Service#Step 6 — Schema evolution & GDPR/PDPA reality|event sourcing's]].

## Step 7 — The scorecard

**Do:** write the trade-off scorecard and the one-paragraph synthesis of what this case adds over Workflows 6–7.

> [!success]- Solution
> **Paid:** enormous storage multiplication (rendition ladder), minutes of publish latency, CDN contracts/complexity, two accuracy classes to explain to stakeholders, client complexity (ABR player).
> **Bought:** Tbps-scale serving on commodity HTTP, rebuffer-free playback on bad networks, transcoding cost ~70% down via preemptible chunks, viral spikes absorbed by caches instead of engineers.
> **Synthesis:** the feed taught read/write asymmetry in *volume*; chat taught it in *statefulness*; video takes asymmetry to the extreme — write and read paths share nothing but a catalog, the serving tier is *someone else's hardware* (CDN), and the biggest architectural wins came from **relocating responsibilities** (intelligence → player, serving → edge, accuracy → batch) rather than scaling anything in place.

---

## Extensions

1. Add live streaming: which parts of the pipeline collapse (no full-file transcode) and what new constraint appears (end-to-end latency budget)? Sketch the LL-HLS variant.
2. Design the recommendation feed for the homepage — recognize it as [[Workflow 6 - Design a News Feed (Twitter-style)|Workflow 6]] wearing a different hat.
3. DRM for a paid tier: license service, encrypted segments — where does the key exchange live, and what does it do to CDN cacheability? (Answer: segments stay cacheable — encrypted; only the tiny license call hits origin.)
4. Cost model: estimate egress cost at 10 Tbps average via CDN contract rates vs cloud egress list prices — and understand why big streamers build their own CDN boxes into ISPs (Netflix Open Connect).
