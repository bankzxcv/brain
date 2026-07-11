---
title: "Workflow 1 - Design Phase of a Real Product"
date: 2026-07-11
tags:
  - software-architecture
  - design-phase
  - adr
  - c4
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 1 — The Design Phase of a Real Product

> [!abstract] Project
> Run the complete design phase for **"KinDee"**, a food-delivery product for a mid-size Thai city: customers order from restaurants, riders deliver, restaurants manage menus. You play the solution architect from kickoff to approved design. No code — the deliverables are the documents real architects produce: characteristics workbook, C4 diagrams, ADRs, walking-skeleton plan.

**Architecture (final target, produced in Step 5):**

```
                          ┌──────────────┐
   Customer app ────────► │              │
   Rider app ───────────► │  API (modular│──────► PostgreSQL (multi-AZ)
   Restaurant portal ───► │  monolith)   │──────► Redis (sessions, hot menus)
                          │              │
                          └──────┬───────┘
                                 │ outbox
                          ┌──────▼───────┐
                          │ Message queue│──► Workers: notifications,
                          └──────┬───────┘    receipts, analytics export
                          ┌──────▼───────┐
                          │ Dispatch svc │◄── rider GPS stream (websocket gw)
                          │ (only real   │
                          │  extraction) │
                          └──────────────┘
   External: payment PSP, maps/routing API, push (FCM/APNs)
```

---

## Step 1 — Extract drivers from the kickoff transcript

The founders tell you: "Launch in Chiang Mai in 5 months. 200 restaurants signed LOIs. We have seed money for 18 months, 6 engineers. Riders are contractors with cheap Android phones. GrabFood exists — our edge is lower commission, so margins are thin. PDPA applies. Lunch is everything: 70% of orders 11:00–13:00."

**Do:** extract (a) functional epics, (b) constraints, (c) candidate architecture characteristics with a measurable target each.

> [!success]- Solution
> **Epics:** ordering & menu browsing, payment, rider dispatch & tracking, restaurant order management, notifications, back-office (support, refunds, reports).
> **Constraints:** 5-month runway to launch; 6 engineers; thin margins → infra frugality; cheap Android → app must be light, tolerate bad networks; PDPA → consent, PII handling, Thai data-residency posture.
> **Characteristics (measurable):**
> | Characteristic | Target | Source |
> |---|---|---|
> | Time-to-market | MVP live in 20 weeks | founder |
> | Elasticity | 6x baseline load 11:00–13:00 daily, autoscale ≤ 3 min | lunch stat |
> | Availability | 99.9% during 09:00–22:00 (windowed!) | order-taking = revenue |
> | Cost efficiency | infra < $1.5k/mo at launch | thin margins |
> | Performance | order placement p99 < 1.5s on 4G | cheap phones, mobile networks |
> | Compliance | PDPA consent + PII audit | law |
>
> Note what did NOT make top tier: global scalability, multi-region, sub-100ms latency. Cutting is the skill.

## Step 2 — Decompose the domain

**Do:** run a lightweight event-storming on paper: list the domain events of one order's life, then group them into bounded contexts. Mark which contexts are core (differentiating), supporting, and generic (buy, don't build).

> [!success]- Solution
> Events: MenuBrowsed → ItemsAdded → OrderPlaced → PaymentAuthorized → OrderAcceptedByRestaurant → FoodReady → RiderAssigned → PickedUp → Delivered → PaymentCaptured → RiderPaidOut...
> **Contexts:** Ordering (cart, order lifecycle) — *core*; Dispatch (rider assignment, tracking) — *core* (the operational edge); Catalog (menus) — supporting; Payments — *generic: integrate a PSP (Omise/2C2P), never build*; Notifications — generic (FCM/LINE); Payouts/Settlement — supporting (spreadsheet-assisted at launch is acceptable!); Identity — generic (managed auth or boring library).
> Strategy: engineering effort concentrates on Ordering + Dispatch; everything generic is bought or kept manual at launch.

## Step 3 — Generate candidate architectures

**Do:** produce three genuinely different candidates for the top characteristics from Step 1, one paragraph each. (No matrix yet.)

> [!success]- Solution
> **A — Modular monolith + queue:** one API deployable (modules: ordering, catalog, dispatch, identity), Postgres, Redis, one queue for async work, dispatch logic in-process. Cheapest, fastest to build.
> **B — Monolith + extracted Dispatch service:** as A, but dispatch (GPS ingest, assignment engine) is its own service consuming events. Rationale: dispatch has *different everything* — write-heavy GPS stream, websockets, geo-queries, spiky, algorithmically volatile.
> **C — Microservices (6–8 services) on Kubernetes:** service per context, event backbone, DB-per-service. Matches the domain map beautifully; costs a platform team the company doesn't have.

## Step 4 — Trade-off matrix & decision

**Do:** score A/B/C against the Step 1 characteristics (weights from business priority), run one sensitivity check ("what if orders 10x in year 1?"), decide.

> [!success]- Solution
> | Criterion (w) | A | B | C |
> |---|---|---|---|
> | Time-to-market (×3) | 5→15 | 4→12 | 2→6 |
> | Elasticity lunch 6x (×2) | 3→6 | 5→10 | 5→10 |
> | Cost (×2) | 5→10 | 4→8 | 2→4 |
> | Perf on 4G (×2) | 4→8 | 5→10 | 4→8 |
> | Team fit (×3) | 5→15 | 4→12 | 2→6 |
> | **Total** | **54** | **52** | **34** |
>
> A edges B — but the sensitivity check matters: GPS ingest (rider pings every 3–5s × hundreds of riders) inside the monolith means the *order API* autoscales because of *location noise* — coupling the revenue path to the noisiest workload. 10x orders → A's score degrades on elasticity and perf; B's doesn't. **Decision: B** — monolith for everything except Dispatch, which is extracted from day one. This is the mature use of the matrix: it localized the *one* extraction that matters; sensitivity analysis broke the tie, not the raw total.

## Step 5 — C4 diagrams

**Do:** draw Context (level 1) and Container (level 2) for architecture B. Mermaid or ASCII.

> [!success]- Solution
> **Level 1 — Context:**
> ```mermaid
> graph TB
>     C[Customer] --> S[KinDee System]
>     R[Rider] --> S
>     M[Restaurant] --> S
>     S --> PSP[Payment provider - Omise]
>     S --> MAPS[Maps/Routing API]
>     S --> PUSH[FCM / LINE Notify]
> ```
> **Level 2 — Container:** (see the diagram at the top of this note) — containers: customer app (Flutter), rider app (Flutter, websocket to dispatch), restaurant portal (web), Core API (modular monolith, Go), Dispatch service (Go, geo-index in Redis), PostgreSQL, Redis, queue (SQS), workers. Annotate every arrow with protocol + sync/async — an arrow without a protocol is a lie waiting to be discovered.

## Step 6 — Write the ADRs

**Do:** write ADR-001 (overall structure) and ADR-002 (dispatch extraction) in full; list titles for ADR-003..006 you'd also write.

> [!success]- Solution
> ADR-001 and ADR-002 follow the template in [[Software Architecture Study#10.2 Architecture Decision Records (ADRs)|Study §10.2]] — write them yourself as the drill; key content: matrix summary, consequences (+ and −), revisit triggers (orders/day > 20k, headcount > 15, second city launch).
> Remaining: **ADR-003** "Payments via PSP redirect, never touch card data" (PCI scope → zero); **ADR-004** "Postgres for everything at launch incl. geo (PostGIS), no specialized stores yet"; **ADR-005** "Order events via transactional outbox" ([[05 - Communication and Data Drills#Problem 9 The outbox problem|why]]); **ADR-006** "Windowed availability target & on-call policy."

## Step 7 — Walking skeleton & risk burn-down

**Do:** define the walking skeleton (thinnest end-to-end slice deployed on real infra) and the 3 riskiest assumptions to attack in weeks 1–4.

> [!success]- Solution
> **Skeleton (2 weeks):** static menu → place order → PSP sandbox payment → restaurant portal shows order → manual "delivered" button → push notification. One module-thin monolith + Postgres on staging + real CI/CD, TLS, monitoring from day one. No dispatch, no GPS.
> **Risk burn-down:** (1) PSP integration friction (Thai PSP sandbox quirks; settlement reports) — prototype first; (2) GPS load reality — 2-day spike: simulate 500 riders pinging, validate Dispatch design + Redis geo-index; (3) restaurant onboarding UX — 200 LOIs mean nothing if menu setup takes 3 hours; test with 5 real restaurants.
> The design phase ends with: characteristics workbook, C4 L1/L2, 6 ADRs, skeleton in production-like env, riskiest assumptions tested. **Architecture continues in every sprint after — this was the beginning, not the whole.**

---

## Extensions

1. Re-run Steps 3–4 assuming 25 engineers and 3 cities — does C win now? What changed the weights?
2. Add a "group ordering" feature: which contexts change? Does it threaten any ADR?
3. Write ADR-007 for switching rider tracking from polling to websockets, including the rejected alternative.
4. Run the same 7 steps for a product you know well at work — the template transfers.
