---
title: "Workflow 10 - Saga Patterns Deep Dive"
date: 2026-07-11
tags:
  - software-architecture
  - saga
  - distributed-transactions
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 10 — Famous Case: Saga Patterns (Distributed Transactions Done Right)

> [!abstract] Project
> Build a travel-booking platform **"Paidee"** (flight + hotel + car + payment across 4 services) three times: first watch 2PC fail, then implement a **choreographed saga**, then an **orchestrated saga**, then harden both against the real world (crashes, duplicates, non-compensatable steps, humans). The saga appeared as a step in [[Workflow 3 - Microservices and Event-Driven on Cloud|W3]] and [[Workflow 9 - Design Ride-Hailing (Uber-style)|W9]]; here it *is* the project. Theory: [[Software Architecture Study#9.2 Saga Pattern|Study §9.2]], warm-up: [[05 - Communication and Data Drills#Problem 4 Design the saga|drills 4–5]].

**Architecture (orchestrated target):**

```
 client ──► Booking API ──► ┌────────────────────────────┐
                            │   Saga Orchestrator        │
                            │  (durable state machine:   │
                            │   saga_id, step, status)   │
                            └─┬────────┬────────┬───────┬┘
                     cmd/reply│        │        │       │
                        ┌─────▼──┐ ┌───▼───┐ ┌──▼───┐ ┌─▼─────┐
                        │ Flight │ │ Hotel │ │ Car  │ │Payment│
                        │  svc   │ │  svc  │ │ svc  │ │ svc   │
                        └────────┘ └───────┘ └──────┘ └───────┘
                        each: local ACID tx + outbox + idempotent handlers
 happy path:   reserve F → reserve H → reserve C → charge → confirm all
 failure path: compensate in REVERSE order (release C → release H → release F)
```

---

## Step 1 — Why not a distributed transaction (2PC)?

**Do:** each service has its own DB ([[Software Architecture Study#8.4 Microservices Architecture|DB-per-service]]). Explain how two-phase commit would work here, then give the three reasons it's rejected in practice — including the one about availability math ([[02 - Quality Attributes Drills#Problem 3 Availability math|drill]]).

> [!success]- Solution
> 2PC: a coordinator asks all 4 services to *prepare* (lock resources, promise to commit), then broadcasts *commit*. Rejected because: (1) **locks held across network calls** — every participant holds row locks for the slowest participant's duration; throughput collapses under load; (2) **coordinator failure between phases leaves participants blocked** holding locks (in-doubt transactions) — the protocol converts one failure into system-wide stall, the opposite of fault isolation; (3) **availability multiplies**: all 4 services *and* the coordinator must be up simultaneously — 0.999⁵ ≈ 99.5%, and worse, most modern infra (message brokers, many cloud DBs, HTTP APIs like a payment PSP) simply doesn't speak 2PC. The saga trades 2PC's atomicity for availability: **each step is a local ACID transaction; global consistency becomes eventual, managed by compensation.**

## Step 2 — Define the saga: steps, compensations, ordering

**Do:** table the Paidee saga: each step's transaction, its compensation, and its *compensatability class* (undoable / retriable / pivot). Then order the steps and justify with the pivot rule.

> [!success]- Solution
> | # | Step (local tx) | Compensation | Class |
> |---|---|---|---|
> | 1 | ReserveFlight (hold seat, 15-min TTL) | ReleaseFlightHold | undoable, cheap |
> | 2 | ReserveHotel (hold room) | ReleaseHotelHold | undoable, cheap |
> | 3 | ReserveCar (hold car) | ReleaseCarHold | undoable, cheap |
> | 4 | **ChargePayment** | RefundPayment (slow, fees, tickets) | **pivot** — expensive to undo |
> | 5 | ConfirmAll (holds → bookings) | — | retriable (must eventually succeed) |
> | 6 | SendItinerary email | — (can't unsend) | retriable, non-compensatable |
>
> **Ordering rules:** (1) cheap-to-undo steps first; (2) the **pivot** (the go/no-go step — here, taking money) as late as possible: before it, abort = free compensations; after it, the saga *must drive forward to completion* (retry confirms forever rather than refund) because backward now costs real money and trust; (3) non-compensatable side effects (email) strictly after the pivot — never apologize for a booking that then fails. This 3-zone shape — *compensatable → pivot → retriable* — is the grammar of every saga.

## Step 3 — Build it as choreography

**Do:** implement the saga with events only, no coordinator ([[Software Architecture Study#9.1 Orchestration vs Choreography|choreography]]). List every event and every subscription each service needs — including the failure paths. Feel the pain accumulate.

> [!success]- Solution
> Happy path: `TripRequested` → Flight: `FlightReserved` → Hotel: `HotelReserved` → Car: `CarReserved` → Payment: `PaymentCaptured` → all three confirm on hearing it.
> Failure subscriptions (the pain): Hotel full → `HotelReservationFailed` → **Flight** must subscribe to release; Car fails → `CarReservationFailed` → **Flight and Hotel** both subscribe; Payment fails → `PaymentFailed` → **all three** subscribe. Every new step multiplies failure-event wiring across all *earlier* services — n steps ≈ O(n²) subscriptions. Nobody knows the saga's state ("where is trip 887?") without correlating events by `saga_id` across 4 services.
> Verdict per the [[04 - Distributed Styles Drills#Problem 6 Broker vs mediator EDA|drill's rule]]: 5 sequential steps + compensations + a pivot = well past choreography's comfort zone (fine for 2–3 steps or pure fan-out). We keep this implementation as the *foil*.

## Step 4 — Build it as orchestration

**Do:** implement the orchestrator: its persisted state machine, the command/reply flow, and crash recovery. Where does the orchestrator's own durability come from, and why is it not a 2PC coordinator in disguise?

> [!success]- Solution
> ```
> saga_instances(saga_id PK, trip_id, current_step, status
>                [RUNNING|COMPENSATING|COMPLETED|FAILED|STUCK], updated_at)
> saga_log(saga_id, step, event, at)   -- append-only audit of every transition
> ```
> Loop: load state → send command (via outbox!) to the step's service → await reply (queue) → persist new state → next. **Crash recovery:** on restart, scan `RUNNING` sagas, re-send the current step's command — safe because every participant is idempotent (Step 5). Timeouts per step → retry with backoff → after N attempts, flip to `COMPENSATING` (before pivot) or keep driving forward / park as `STUCK` (after pivot).
> **Not 2PC because:** participants hold **no locks** waiting for others — each local tx commits immediately (the hotel hold is a *business-level* reservation with a TTL, not a DB lock); the orchestrator being down pauses *progress*, never blocks *participants*; and consistency is eventual-by-contract, with intermediate states (`PENDING` bookings) visible and modeled, not hidden behind in-doubt locks.

## Step 5 — Idempotency, duplicates, and the outbox (the plumbing that makes it true)

**Do:** wire the three reliability patterns every saga step needs: exactly-once *effect* on redelivered commands, atomic state+message updates, and correlated replies. (All three are drills you've done — compose them.)

> [!success]- Solution
> 1. **Idempotent handlers** ([[04 - Distributed Styles Drills#Problem 7 Idempotent consumer|drill]]): command carries `saga_id + step` as idempotency key; participant records it in the same tx as the effect → redelivery returns the *recorded prior reply* (not an error — the orchestrator needs the answer again).
> 2. **Outbox everywhere** ([[05 - Communication and Data Drills#Problem 9 The outbox problem|drill]]): participant commits `{business effect + processed-key + reply-message}` in ONE local tx; relay publishes. Orchestrator does the same for commands + state. No dual writes anywhere = no lost/phantom steps.
> 3. **Correlation:** every command/reply carries `saga_id`, `step`, `attempt`; replies go to the orchestrator's reply queue ([[05 - Communication and Data Drills#Problem 3 Request/reply over messaging|request/reply drill]]). The `saga_log` + correlation IDs are also what [[Workflow 11 - OpenTelemetry - Logs Traces Metrics|observability]] will hang traces off.
> With these three, the saga survives: message redelivery, participant crash mid-step, orchestrator crash mid-saga, and relay lag — the whole failure menagerie — with no manual repair.

## Step 6 — The ugly cases: humans, deadlines, and half-failures

**Do:** design the handling for: (a) compensation itself fails (hotel API down during release), (b) saga stuck after the pivot (charged, but confirm keeps failing), (c) the 15-min flight hold expires mid-saga, (d) rider cancels while the saga is running.

> [!success]- Solution
> - (a) Compensations are **retriable commands** like any step (idempotent, backoff); after N failures → DLQ + `STUCK` + alert. The saga_log tells the on-call human *exactly* what to fix and where to resume — sagas don't eliminate manual intervention, they make it **bounded and legible**.
> - (b) Post-pivot rule: drive forward. Retry confirm for hours (the money is taken; the customer wants the trip, not a refund). If truly impossible (hotel overbooked): *business* decision — auto-refund + voucher, i.e., a **compensation defined by product policy**, echoing [[Workflow 9 - Design Ride-Hailing (Uber-style)#Step 5 — The trip state machine & saga|W9's collections case]].
> - (c) TTL expiry = a participant-side timeout event `FlightHoldExpired` → orchestrator treats it as step failure → compensate others or re-reserve (re-run step 1) if still pre-pivot. Deadline pressure is why holds exist at all — model time as events, not as hope.
> - (d) Cancellation = a *competing command*: orchestrator checks state — pre-pivot: flip to `COMPENSATING`, done; mid-step: mark `CANCEL_REQUESTED`, apply after the in-flight reply arrives (never interrupt a step); post-pivot: it's a refund workflow, a *new* saga. Cancellation-as-saga-input separates "user intent" from "system progress" — the cleanest mental model.

## Step 7 — Framework or not, and the scorecard

**Do:** compare hand-rolled (what you built) vs Temporal/AWS Step Functions for this saga. Then the scorecard + the one-paragraph rule for when a saga is even warranted.

> [!success]- Solution
> | | Hand-rolled | Temporal-style | Step Functions |
> |---|---|---|---|
> | State persistence, retries, timers | you build (Steps 4–5) | ✅ platform | ✅ platform |
> | Saga as readable code | scattered | ✅ one workflow function | ASL JSON (clunky branching) |
> | Ops | your DB + queues | run/buy a cluster | ✅ serverless, AWS-only |
> | Lock-in / testability | none / yours | SDK coupling / excellent replay-testing | AWS / moderate |
>
> Hand-roll to *learn* (you now know what the frameworks solve); adopt a platform when saga count > ~3 or steps > ~5 — undifferentiated heavy lifting ([[06 - Trade-off Analysis Drills#Problem 4 Buy vs build vs managed|buy-vs-build drill]]).
> **Scorecard — paid:** every step needs idempotency+outbox ceremony; intermediate states leak into UX and support; compensations are product decisions in disguise; testing = failure-injection matrices. **Bought:** no distributed locks, per-service autonomy intact, availability compounds instead of multiplying, every failure bounded + auditable.
> **When warranted:** only when a workflow spans **multiple ownership/transaction boundaries** *and* partial completion has real cost. If all steps live in one DB — use one ACID transaction and go home ([[Workflow 2 - Modular Monolith on a Cloud VM#Step 2 — Wire one real cross-module flow|W2's gift]]); a saga where a transaction would do is self-inflicted distributed systems.

---

## Extensions

1. Add a 5th step: loyalty-points award. Which zone does it belong to (compensatable / pivot / retriable), and does it move the pivot?
2. Implement the *parallel* variant: reserve flight+hotel+car concurrently, then charge. What changes in compensation logic and failure counting? When is parallel worth it (latency budget vs complexity)?
3. Write the failure-injection test matrix: {each step} × {timeout, error-reply, duplicate-reply, crash-after-effect-before-reply} — automate 3 of them.
4. Map [[Workflow 5 - Solution Architecture Case Study with Trade-offs|W5's claims submission]] onto this template: identify its pivot and its non-compensatable steps.
