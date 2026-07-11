---
title: "Workflow 9 - Design Ride-Hailing (Uber-style)"
date: 2026-07-11
tags:
  - software-architecture
  - system-design
  - geospatial
  - ride-hailing
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 9 — Famous Case: Ride-Hailing (Uber-style)

> [!abstract] Project
> Design ride matching at city scale: riders request, nearby drivers get offered the job, one accepts, both track each other live, money moves at the end. [[Workflow 1 - Design Phase of a Real Product|Workflow 1's KinDee]] touched dispatch at startup scale — this is the same domain at "famous interview question" scale, adding: **geospatial indexing**, **a real-time matching market**, and a **state machine that spans two humans moving through physical space**.

**Architecture (target):**

```
 rider app ──► API gateway ──► Trip service (state machine + saga)
 driver app ══ws══► Location gateway                │
        (pings every 4s)      │              ┌──────┴──────┐
                              ▼              ▼             ▼
                    Location service    Matching svc   Pricing svc
                    in-memory geo-      (per-city      (surge: demand/
                    index (geohash/H3   matching loop)  supply per cell)
                    cells, sharded      │
                    by city/region)     ├─► offer → driver (via ws) → accept/timeout
                              │         └─► assign → Trip service
                    ETA/Maps service (routing engine, traffic data)
                    Trips DB (state) · Payments (end of trip, W4-style ledger)
                    Analytics stream: every ping/event → Kafka → surge, fraud, data science
```

---

## Step 1 — Requirements & what's genuinely hard

**Do:** scope (request, match, track, pay; skip pooling/food). 5M drivers online at peak globally, pinging every 4s; 1M ride requests/hour peak. Compute the ping load, then rank what's hard: is it the writes, the matching, or the consistency?

> [!success]- Solution
> Pings: 5M / 4s = **1.25M location writes/s** — dwarfing every other write in the system (rides are a rounding error: ~280/s). But each ping is tiny, only the *latest few* matter, and losing one is harmless.
> **Ranking the hard parts:** (1) *matching correctness under contention* — two riders must not be assigned the same driver (real consistency problem); (2) *location ingest volume* — huge but forgiving (lossy is fine); (3) *the trip state machine* — long-lived (20+ min), crosses two devices on mobile networks, and ends in money. Characteristics: matching latency (< 30s to a confirmed driver — the product moment), correctness of assignment (exactly-one), location freshness (~5s), availability per-city (a city down = local news). Note how *different data classes get different guarantees* — the recurring lesson of [[Workflow 7 - Design a Chat System (WhatsApp-style)#Step 6 — Presence, receipts, and typing indicators|W7]] and [[Workflow 8 - Design Video Streaming (YouTube-style)#Step 5 — Metadata, search, and view counts|W8]], now driving the whole design.

## Step 2 — Location ingest: the firehose with amnesia

**Do:** design the path for 1.25M pings/s. Why is a relational DB the wrong sink? Where do pings actually go (hint: two places with different lifespans)?

> [!success]- Solution
> Drivers hold a websocket/UDP-ish channel to regional **location gateways** ([[Workflow 7 - Design a Chat System (WhatsApp-style)#Step 2 — Why not HTTP? Choosing the connection model|W7's connection layer]], reused). A relational DB at 1.25M writes/s of data that's stale in 4 seconds is paying durability prices for data with a 4-second life — the presence anti-pattern at 100× scale.
> Two sinks: (1) **in-memory geo-index** (the matching truth): sharded by city/region, holds only `driver_id → (cell, lat/lng, heading, status, ts)` — overwrite-in-place, no history, rebuildable from live pings in seconds after a crash (cache-of-truth-held-by-clients, like W7's registry); (2) **Kafka stream** (the analytical truth): every ping appended for surge computation, ETA models, fraud, replay — consumers are all async and can lag. Nothing durable sits on the hot path.

## Step 3 — Geospatial indexing: "drivers near me"

**Do:** you need "available drivers within ~2km of point P" in <10ms, continuously updating. Why do B-tree indexes fail? Explain the geohash/H3 cell approach and the boundary problem.

> [!success]- Solution
> A B-tree indexes one dimension; `lat BETWEEN … AND lng BETWEEN …` scans a band of the earth. **Cell schemes (geohash/S2/H3)** map 2D → 1D: divide the world into hierarchical cells; each driver's ping updates their cell set membership (`cell → {drivers}` in memory). Query = look up P's cell + its neighbors (fixes the **boundary problem** — a driver 50m away across a cell edge — always search the ring), then exact-distance filter the small candidate set. O(candidates-in-3km²), not O(drivers-on-earth).
> Cell size trade-off: small cells = more precise, more membership churn; big cells = fewer updates, more false candidates to filter — pick per zoom level (H3 res ~8–9 for city matching). Sharding follows geography (city/region owns its cells) — natural, because **rides never span shards**: the rare cross-boundary case (airport runs between cities) is handled by querying both regions, not by globalizing the index.

## Step 4 — Matching: the contention moment

**Do:** design the match: rider requests → candidate drivers → offer → accept. Two riders, one driver, same second: what prevents double-assignment? Compare lock-based vs single-writer designs ([[Workflow 4 - CQRS and Event Sourcing Service#Step 2 — Build the event store on Postgres|W4's OCC]] is a cousin).

> [!success]- Solution
> Flow: Trip service → Matching service (per-city): query geo-index → rank candidates (ETA, driver rating, acceptance rate) → **offer to the best driver with a 10–15s TTL** → accept wins / timeout cascades to next candidate (or small parallel batch with first-accept-wins — trades driver annoyance for latency).
> **Double-assignment prevention:** the driver's status transition `available → offered → assigned` must be atomic. Two clean designs: (a) **single-writer per city** — the city's matching loop is one logical process (sharded by region); all offers for that city serialize through it: no locks because no concurrency, Kafka-partition-style; (b) **atomic CAS** on driver status in the shared store (`SET status=offered IF status=available`) — optimistic, like W4's OCC, retry on conflict. At city scale (a): the matching *market* benefits from a global view (batching nearby requests for better assignments), and one process per city is nowhere near a compute ceiling. Big insight: **partition by the natural contention boundary (the city), then be gloriously single-threaded inside it.**

## Step 5 — The trip state machine & saga

**Do:** model the trip lifecycle end-to-end (request → … → paid) as a state machine. Where does it live, what drives transitions, and how does the end-of-trip payment reuse [[05 - Communication and Data Drills#Problem 4 Design the saga|the saga drill]]?

> [!success]- Solution
> ```
> REQUESTED → MATCHING → OFFERED → ACCEPTED → DRIVER_ARRIVING
>   → IN_PROGRESS → COMPLETED → PAYMENT_PENDING → PAID
> (+ CANCELLED_BY_RIDER/DRIVER, NO_DRIVERS, PAYMENT_FAILED from most states)
> ```
> Lives in the **Trip service** — durable (Postgres row per trip + event log of transitions; the trip is audit-worthy, unlike pings), transitions driven by app events (driver taps "arrived"), GPS inference (auto-detect pickup), and timeouts (offer expiry, driver no-show). Every transition validated against the current state (no `PAID` from `MATCHING`) and emitted to Kafka — receipts, analytics, and support tools are projections.
> End of trip = **orchestrated saga**: compute final fare (pricing svc) → charge PSP → payout ledger entry (a [[Workflow 4 - CQRS and Event Sourcing Service|W4-style wallet]]!) → receipt. Compensations: retry card → fallback payment → debt-flag on rider account (ride already happened — *some steps are physically non-compensatable*, so the saga degrades to collections, a business process; the drill's "order risky steps last" meets its limit here and the mitigation is product policy, not code).

## Step 6 — Surge pricing & ETA: the stream-fed brains

**Do:** design surge (per-cell price multiplier from supply/demand) and ETA. Why do both *have* to be built on the Kafka ping stream rather than the live index?

> [!success]- Solution
> **Surge:** streaming job windows the last ~5 min of pings + requests per H3 cell → supply/demand ratio → multiplier table (cell → 1.0–3.0×), pushed to pricing svc + shown in apps. Needs *short history and trends* (demand rising?) — exactly what the amnesiac live index doesn't have and the stream does. Update cadence ~30s; nobody needs surge to be transactional (it's a *price quote*, locked in at request time — the locked quote is the consistency boundary, another product-rule-as-architecture).
> **ETA:** routing engine over road graph + live traffic derived from… the same ping stream (drivers *are* the traffic sensors — the firehose becomes the company's proprietary data asset). Precompute per-road-segment speeds continuously; per-request ETA = graph search with current weights. Both are textbook [[Software Architecture Study#8.5 Event-Driven Architecture (EDA)|EDA]] consumers: added later without touching ingest — the extension superpower from [[Workflow 3 - Microservices and Event-Driven on Cloud|W3]].

## Step 7 — Failure drills & scorecard

**Do:** predict behavior: (a) location service shard (one region) crashes, (b) matching loop for a city dies mid-offer, (c) rider's phone dies mid-trip, (d) PSP outage at trip end. Then the scorecard + what this case adds to the course.

> [!success]- Solution
> - (a) In-memory index gone → rebuilt from live pings in ~2 ping-cycles (~10s); matching pauses briefly in that region; trips in progress unaffected (Trip service is separate + durable). Designed-for amnesia pays off.
> - (b) Offers have TTLs and the state machine is durable: watchdog restarts the loop, in-flight offers expire naturally, requests re-enter matching. Worst case: a driver accepted as the loop died → accept hits Trip service (the durable arbiter), which either confirms or rejects-with-apology — the state machine, not the matcher, owns truth.
> - (c) Trip continues on driver-side events + GPS; rider reconnects and syncs state ([[Workflow 7 - Design a Chat System (WhatsApp-style)#Step 3 — The message flow A sends to B|pull-as-truth]] again); payment completes regardless (card on file).
> - (d) Trip → `PAYMENT_PENDING`, saga retries with backoff for hours; rider owes; both parties see honest status. Never block trip completion on the PSP ([[04 - Distributed Styles Drills#Problem 2 Fallacy autopsy|fallacy #1]]).
> **Scorecard — paid:** per-city operational surface, in-memory infra + rebuild logic, a state machine with dozens of edges, stream platform as a hard dependency for pricing. **Bought:** 1.25M writes/s on commodity memory, <10ms geo-queries, exactly-one assignment without distributed locks, surge/ETA/fraud as free-riding stream consumers.
> **What it adds:** geospatial partitioning as *the* sharding strategy, contention solved by **partition + single-writer** rather than locks, and the clearest example yet of guarantee-tiering: pings (lossy) < surge (approximate) < trip state (durable) < money ([[Workflow 4 - CQRS and Event Sourcing Service|ledger-grade]]) — four data classes, four price points, one product.

---

## Extensions

1. Add ride-pooling (shared rides): matching becomes a routing optimization over *future* trajectories — which components change, and why does matching latency budget explode?
2. Driver incentives ("complete 20 trips → bonus") — design the counting pipeline; decide which accuracy class it needs (hint: it's money-adjacent).
3. Simulate the airport rush: 2,000 riders, 300 drivers, one H3 cell. Does your matching design queue fairly? What does surge do to the queue? Write the trade-off note on fairness vs revenue.
4. Do the full [[06 - Trade-off Analysis Drills#Problem 8 Full drill — timed solution architecture|timed SA drill]] treating this as a *new market entry*: same product, 1 city, 6 engineers — how much of this architecture do you *not* build on day one? (Compare your answer to [[Workflow 1 - Design Phase of a Real Product|Workflow 1]].)
