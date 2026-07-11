---
title: "05 - Communication and Data Drills"
date: 2026-07-11
tags:
  - software-architecture
  - messaging
  - cqrs
  - saga
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 05 — Communication & Data Drills

> [!abstract] What you're drilling
> Sync vs async decisions, events vs commands, sagas & compensation, CQRS, event sourcing. See [[Software Architecture Study#8.0 Synchronous & Asynchronous Communication|Study §8.0]] and [[Software Architecture Study#9. Coordination & Data Patterns|§9]].

---

### Problem 1: Sync or async?

For each interaction, choose sync or async and give the deciding factor: (a) checking card validity during checkout, (b) sending the order-confirmation email, (c) fetching the user's profile to render a page, (d) recalculating "customers also bought" after a purchase, (e) reserving inventory during checkout, (f) syncing orders to the data warehouse.

**Expected output:**
```
6 choices + deciding factors.
```

> [!hint]- Hint
> Does the caller need the answer *now* to proceed?

> [!success]- Solution
> - (a) **Sync** — checkout can't proceed without the result.
> - (b) **Async** — user needn't wait; retries handle email-provider blips.
> - (c) **Sync** — the page needs the data (consider cache).
> - (d) **Async** — analytics; minutes of lag are invisible.
> - (e) **Sync** (or orchestrated saga step) — overselling is a business error; the user must know now.
> - (f) **Async** — batch/stream; warehouses are eventually consistent by nature.
> Pattern: *user-blocking + needs-answer* → sync; *side effect* → async.

### Problem 2: Event or command?

Rewrite these poorly-named messages as proper events or commands and say who should own each decision: (a) topic `send-welcome-email` published by UserService, (b) queue message `UserCreated` sent directly to EmailService's queue, (c) `OrderService` publishes `ReserveInventory` to a topic consumed by whoever.

**Expected output:**
```
Corrected message types + ownership reasoning.
```

> [!hint]- Hint
> Events = past fact, broadcast, producer indifferent. Commands = imperative, one addressee, sender cares.

> [!success]- Solution
> - (a) UserService shouldn't know emails exist. Publish event **`UserRegistered`**; EmailService subscribes and decides to send a welcome email. (Decision moves to the consumer.)
> - (b) Naming an event but addressing one consumer = disguised command. Either publish `UserRegistered` to a topic (true event) or send command **`SendWelcomeEmail`** to EmailService's queue — pick the semantic, don't mix.
> - (c) `ReserveInventory` is imperative — a command "consumed by whoever" is dangerous (who's responsible? what if two consumers act?). Send it to InventoryService explicitly, or publish event `OrderPlaced` and let InventoryService own the reservation decision.

### Problem 3: Request/reply over messaging

Checkout (sync HTTP to the user) must get a fraud score from a fraud service that only consumes from a queue. Design request/reply over messaging: what goes in the message, how does the reply find its way back, and what happens on timeout?

**Expected output:**
```
Message fields, reply routing, timeout policy.
```

> [!hint]- Hint
> Correlation ID + reply-to queue.

> [!success]- Solution
> Request message: `{correlationID, replyTo: "checkout-replies-<instance>", orderPayload}`. Fraud service processes and publishes the score to the `replyTo` queue with the same `correlationID`. Checkout holds the HTTP request, awaiting the correlated reply (in-memory map of pending correlation IDs) with a **timeout (e.g., 2s)** → on timeout: degrade per policy — e.g., approve low-value orders / hold high-value for review (*fail-open vs fail-closed is a business decision, record it as an ADR*). Note: if you need this pattern often, ask whether the fraud service should just expose sync gRPC.

### Problem 4: Design the saga

Trip booking = reserve flight + reserve hotel + charge card, across 3 services. Design an **orchestrated saga**: list the steps, the compensating transaction for each, and the correct *order* of steps given that one action is hardest to compensate.

**Expected output:**
```
Step table with compensations + ordering rationale.
```

> [!hint]- Hint
> Do the riskiest-to-undo step last.

> [!success]- Solution
> | Step | Action | Compensation |
> |---|---|---|
> | 1 | ReserveFlight (hold) | ReleaseFlightHold |
> | 2 | ReserveHotel (hold) | ReleaseHotelHold |
> | 3 | ChargeCard | RefundCharge |
> | 4 | ConfirmFlight + ConfirmHotel | — (post-payment confirm) |
>
> **Ordering:** money moves **last** — refunds are slow, cost fees, and generate support tickets, while releasing holds is free and instant. Orchestrator (e.g., Temporal/Step Functions state machine) drives steps, persists saga state, retries transient failures (idempotent steps! see [[04 - Distributed Styles Drills#Problem 7 Idempotent consumer|dedupe drill]]), and runs compensations in reverse on failure. Note compensations can fail too → retry + park in DLQ + alert a human.

### Problem 5: Choreograph it instead

Redesign Problem 4 as a **choreographed saga** (events only). Show the event chain including the failure path when the card charge fails. Then state honestly what got worse.

**Expected output:**
```
Event chain (happy + failure) + 2 honest downsides.
```

> [!hint]- Hint
> Every service must listen for downstream failures to undo itself.

> [!success]- Solution
> Happy: `TripRequested` → Flight reserves, emits `FlightReserved` → Hotel reserves, emits `HotelReserved` → Payment charges, emits `PaymentCaptured` → Flight & Hotel confirm.
> Failure: Payment emits `PaymentFailed` → **both** Flight and Hotel must subscribe to it and release their holds; each emits `FlightReleased`/`HotelReleased`.
> **Worse:** (1) workflow state is nowhere — "where is trip 123 stuck?" requires correlating events across 3 services (need correlation IDs + tracing); (2) failure logic is smeared: adding a car-rental step means teaching *it* about every failure event, and teaching payment-failure handling to it. For sequential+compensating flows, orchestration was the better call — this drill is the proof.

### Problem 6: CQRS — pick the right level

For each system, choose CQRS level 0 (none), 1 (separate models, same DB), 2 (read replicas), or 3 (separate read store): (a) startup CRUD todo app, (b) e-commerce with product pages read 500:1 vs writes, (c) analytics dashboard over an OLTP order DB, slow queries killing checkout, (d) social feed needing full-text search + personalization.

**Expected output:**
```
4 level choices + reasoning.
```

> [!hint]- Hint
> Use the lightest level that solves the actual problem.

> [!success]- Solution
> - (a) **0** — CQRS on a todo app is resume-driven development.
> - (b) **2** — replicas + cache absorb read volume; same shape data, no need for a new store.
> - (c) **2→3** — first move analytics to a replica (isolates OLTP now); grow to a warehouse/columnar store (3) when query shapes diverge.
> - (d) **3** — search needs Elasticsearch/OpenSearch, feed needs precomputed per-user views; write model stays normalized, projections build both read stores from events/CDC.

### Problem 7: The stale-read UX problem

With CQRS level 3, a user edits their shipping address, the page reloads from the read store, and shows the *old* address (projection lag ~1s). Users file bugs. Give 3 architectural/UX remedies without abandoning CQRS.

**Expected output:**
```
3 remedies with trade-offs.
```

> [!hint]- Hint
> Read-your-own-writes is the requirement; global freshness is not.

> [!success]- Solution
> 1. **Optimistic UI:** render the submitted value from the client immediately; reconcile in background. Cheap, standard, hides ~seconds of lag.
> 2. **Read-your-writes routing:** after a write, serve *that user's* reads from the write model (or pin to the write DB) for N seconds / until projection version ≥ write version (version token returned by the command). Precise, more plumbing.
> 3. **Synchronous projection for critical views:** update the specific read view in the same transaction/request for high-stakes data (addresses, balances), keep async for the rest. Erodes some CQRS decoupling — apply narrowly.

### Problem 8: Event-source or not?

Two candidates: (A) a wallet/ledger service (deposits, withdrawals, balance disputes, regulator audits); (B) a user-profile service (name, avatar, preferences). For each: event-source or not, and why? For the one you event-source, list the event types and how `GetBalance` works.

**Expected output:**
```
2 decisions + event list + balance query design.
```

> [!hint]- Hint
> Event-source domains that *are* ledgers.

> [!success]- Solution
> **(A) Yes** — a wallet *is* an event log; disputes and audits need the exact history; replay enables new projections (fraud patterns).
> Events: `WalletOpened`, `FundsDeposited{amount, ref}`, `FundsWithdrawn`, `WithdrawalRejected{reason}`, `FundsHeld/Released` (disputes).
> `GetBalance`: never replay-on-read in the hot path — maintain a **projection** (current-balance table updated by the event stream) or **snapshot + tail replay**; the event log stays the source of truth, the projection is disposable.
> **(B) No** — nobody audits avatar history; current-state CRUD with an `updated_at` is correct. Event-sourcing it buys schema-versioning-forever pain for zero business value.

### Problem 9: The outbox problem

OrderService must (1) commit the order to its DB and (2) publish `OrderPlaced` to Kafka. A naive implementation does the DB commit, then the publish — and crashes between them. Name the problem and design the standard fix.

**Expected output:**
```
Problem name + outbox pattern design.
```

> [!hint]- Hint
> You can't atomically commit to a DB *and* a broker.

> [!success]- Solution
> **Dual-write problem** — DB and broker can't share a transaction; crash between them = order exists but no event (or worse, reversed order = event without order).
> **Transactional outbox:** in the *same DB transaction* as the order insert, insert the event into an `outbox` table. A relay (poller or Debezium CDC) reads the outbox and publishes to Kafka, marking rows sent. Guarantees at-least-once publication (consumers dedupe — [[04 - Distributed Styles Drills#Problem 7 Idempotent consumer|idempotency drill]]). The DB transaction is the single source of atomicity.

### Problem 10: End-to-end integration drill

Design the full data/communication plan for "user places an order" in a microservices shop, combining everything: which calls are sync, where the saga sits, where the outbox sits, which events fan out, and where CQRS serves reads. One diagram + one paragraph.

**Expected output:**
```
Annotated flow diagram + narrative.
```

> [!hint]- Hint
> Sync facade → orchestrated core → choreographed periphery → projected reads.

> [!success]- Solution
> ```
> client ──sync HTTP──► API GW ──► OrderService
>   OrderService: tx { insert order; outbox(OrderPlaced) } ──202 Accepted + orderID──► client
>   relay ──► broker: OrderPlaced
>   CheckoutOrchestrator (saga): cmd→Payment → cmd→Inventory → cmd→Shipping
>        compensations: Refund / Release on failure; final event: OrderConfirmed | OrderFailed
>   OrderConfirmed fan-out (choreography): Email svc, Loyalty svc, Analytics, Search-indexer
>   Projections: OrderConfirmed/Failed → order-status read store ◄──sync GET /orders/42── client (polling or push)
> ```
> Narrative: the user-facing write is a short sync call that only persists intent (order + outbox atomically), returning immediately. The money-and-stock workflow runs as an orchestrated saga (visibility + compensation). Everything that merely *reacts* to the outcome is choreographed fan-out. Reads come from a projection, so the order-status page never touches the write path — and the UI shows "processing" honestly while the saga runs (eventual consistency surfaced as UX, not hidden).
