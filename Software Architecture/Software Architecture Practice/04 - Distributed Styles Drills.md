---
title: "04 - Distributed Styles Drills"
date: 2026-07-11
tags:
  - software-architecture
  - microservices
  - event-driven
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 04 — Distributed Styles Drills

> [!abstract] What you're drilling
> Client/server, n-tier, service-based, microservices, event-driven, space-based, serverless — boundaries, granularity, and failure thinking. See [[Software Architecture Study#8. Distributed (Multiple-Application) Styles|Study §8]].

---

### Problem 1: Tiers vs layers

Your layered monolith runs entirely on one VM. Ops proposes: move the DB to its own server, later add a load balancer + second app VM. After both steps, how many tiers and how many layers? Explain the difference in one sentence.

**Expected output:**
```
Tier/layer counts + the distinction.
```

> [!hint]- Hint
> Layers are logical; tiers are physical.

> [!success]- Solution
> Still **4 layers** (logical code organization unchanged); now **2 tiers** (app tier ×2 VMs + data tier). Layers describe how code is organized inside a deployable; tiers describe physically separated deployment units — you changed deployment, not architecture style.

### Problem 2: Fallacy autopsy

For each incident, name the violated fallacy of distributed computing: (a) checkout hangs 30s when the recommendation service is slow, (b) the batch job saturates the office VPN and voice calls drop, (c) a service in Singapore calls one in Virginia per request, users complain, (d) after a switch replacement, service discovery breaks, (e) intern reads unencrypted service-to-service traffic on the pod network.

**Expected output:**
```
5 fallacy names + the fix for each.
```

> [!hint]- Hint
> Reliability, bandwidth, latency, topology, security.

> [!success]- Solution
> - (a) "The network is reliable" → timeouts (e.g., 300ms) + circuit breaker + graceful degradation (show generic recs).
> - (b) "Bandwidth is infinite" → throttle/schedule the job, compress, dedicated link.
> - (c) "Latency is zero" → co-locate chatty services in one region; cache; batch calls.
> - (d) "Topology doesn't change" → discovery must handle network churn; health-check-driven registration, not static IPs.
> - (e) "The network is secure" → mTLS between services (mesh or app-level), zero-trust posture.

### Problem 3: Service-based as the middle path

A 15-dev company runs a struggling e-commerce monolith: deploys take 2h and break weekly; checkout and catalog teams block each other. The CTO says "microservices!" Propose service-based architecture instead: define the services, the shared-DB rule, and 3 reasons it beats full microservices *for them*.

**Expected output:**
```
4–6 coarse services + DB strategy + 3 reasons.
```

> [!hint]- Hint
> Their pain is deploy coupling and team collision — not scale.

> [!success]- Solution
> **Services:** Storefront/Catalog, Checkout/Orders, Payments, Fulfillment, Admin. Each independently deployable; **one shared Postgres** but each service owns a schema namespace and only writes its own tables (fitness-function enforced).
> **Why not micro:** (1) their pain is deploy coupling — 5 coarse services already fix it without 25 pipelines; (2) shared DB keeps ACID transactions for order+payment+inventory — no sagas needed at their scale; (3) 15 devs can't staff the platform team (K8s, tracing, service mesh) microservices demand. Migration path preserved: schemas-per-service can later become DBs-per-service.

### Problem 4: Draw the microservices baseline

From memory: draw a minimal microservices deployment for a food-delivery app (Orders, Restaurants, Couriers, Payments) including the gateway, per-service DBs, and one async path. Then list the 6 supporting components you must budget for that aren't services.

**Expected output:**
```
ASCII diagram + 6-item platform list.
```

> [!hint]- Hint
> Nothing shares a database. Delivery assignment is naturally async.

> [!success]- Solution
> ```
>                 ┌─────────────┐
>  mobile/web ──► │ API Gateway │ (authn, rate limit, routing)
>                 └─┬───┬───┬───┘
>       ┌──────────┘   │   └───────────┐
>  ┌────▼───┐    ┌─────▼────┐    ┌─────▼────┐
>  │ Orders │    │Restaurant│    │ Payments │
>  └─┬────┬─┘    └────┬─────┘    └────┬─────┘
>  ┌─▼──┐ │ OrderPlaced          ┌────▼───┐
>  │OrdDB│ └───►[broker]         │ PayDB  │
>  └────┘        │               └────────┘
>          ┌─────▼─────┐   ┌────────┐
>          │ Couriers  │──►│CourDB  │  (assigns delivery async)
>          └───────────┘   └────────┘
> ```
> **Platform budget:** CI/CD per service, container orchestration (K8s/ECS), centralized logging, distributed tracing + metrics, service discovery, secrets management. (Also: contract testing, DLQs.)

### Problem 5: Granularity referee

Two proposals for the Orders domain: (A) one OrderService; (B) OrderCreationService, OrderValidationService, OrderPricingService, OrderHistoryService — "single responsibility!" Referee using granularity drivers: which disintegration drivers justify splitting, which integration drivers argue for merging, and what's your verdict?

**Expected output:**
```
Driver analysis + verdict.
```

> [!hint]- Hint
> Creation/validation/pricing share one transaction. History doesn't.

> [!success]- Solution
> **Split drivers present?** Volatility: pricing rules change weekly (weak yes). Scale: history reads 50x order writes (yes). Fault/security: no. **Merge drivers:** creation+validation+pricing form **one workflow, one transaction** — splitting them means distributed transactions and constant chatter (the distributed-monolith smell).
> **Verdict:** OrderService (create/validate/price together) + optionally OrderHistoryService (separate read-scaling, no shared transaction — really a [[Software Architecture Study#9.3 CQRS — Command Query Responsibility Segregation|CQRS]] read model). Single-responsibility applies to *cohesion of the domain*, not "one verb per service."

### Problem 6: Broker vs mediator EDA

An "OrderPlaced" event must trigger: payment capture → (on success) inventory reservation → shipping label → customer email. Payment failures must refund/cancel everything. Design it twice — broker topology and mediator topology — then pick one and defend it.

**Expected output:**
```
Two event-flow sketches + a defended choice.
```

> [!hint]- Hint
> Count the steps, branches, and compensations.

> [!success]- Solution
> **Broker:** `OrderPlaced` → Payment listens, emits `PaymentCaptured`/`PaymentFailed` → Inventory listens to Captured, emits `InventoryReserved`/`OutOfStock` → Shipping listens… failure compensations = every service also listening to downstream failure events (Payment must hear `OutOfStock` to refund). Workflow logic smeared across 4 services.
> **Mediator:** CheckoutOrchestrator receives `OrderPlaced`, commands Payment → Inventory → Shipping in sequence, tracks state, runs compensations (refund, release) on any failure.
> **Choice: mediator.** Sequential steps + branching + mandatory compensation = orchestration's home turf ([[Software Architecture Study#9.1 Orchestration vs Choreography|Study §9.1]]). Broker topology shines for the *fan-out* parts — email/analytics/loyalty subscribing to `OrderCompleted` — so use both: orchestrate the transaction, choreograph the notifications.

### Problem 7: Idempotent consumer

Your payment consumer receives `CapturePayment{orderID: 42, amount: 500}` twice (at-least-once delivery). Design the dedupe: what do you key on, where do you store it, and what does the consumer do on a duplicate? What must the *producer* include to make this possible?

**Expected output:**
```
Dedupe design (key, store, behavior) + producer obligation.
```

> [!hint]- Hint
> The check and the business effect must be atomic.

> [!success]- Solution
> Producer includes a **unique, stable message ID** (or idempotency key = e.g., `orderID+attempt`). Consumer, **in one DB transaction**: `INSERT INTO processed_messages(msg_id)` (PK) + apply the payment effect. Duplicate → PK violation → ack the message, skip the effect, log. Storing the dedupe record outside the same transaction reintroduces the race. TTL the table by retention window. Alternative: make the effect naturally idempotent (`UPDATE orders SET status='paid' WHERE id=42 AND status='pending'`).

### Problem 8: Space-based or not?

Concert-ticket sale: 500k users hit "buy" in the first minute for 20k seats; overselling is unacceptable. Explain why plain n-tier + relational DB struggles, sketch the space-based approach, and identify the one part of the problem space-based makes *harder*.

**Expected output:**
```
Bottleneck analysis + sketch + the hard part.
```

> [!hint]- Hint
> Elasticity is solved; what about 500k people contending for the *same rows*?

> [!success]- Solution
> N-tier: app tier scales out fine but every purchase hammers the same `seats` rows — lock contention and connection limits melt the DB.
> **Space-based:** requests hit processing units holding seat inventory in replicated in-memory grids; writes async-persist to the DB later; units scale with load.
> **The hard part:** *consistency on high contention.* 20k seats replicated across units → replication lag = oversell risk — exactly the unacceptable outcome. Real designs partition inventory (each unit *owns* a seat range, no replication of hot data) or route all writes for a seat block to one owner (turning it into a sharded, single-writer design). Space-based buys elasticity; it does not repeal contention.

### Problem 9: Serverless judgment call

Classify each workload as great / poor fit for FaaS (Lambda), with the deciding factor: (a) image thumbnailing on S3 upload, (b) sustained 3k RPS core API, (c) nightly report generation, (d) WebSocket chat server, (e) Stripe webhook handler, (f) 45-minute video transcode.

**Expected output:**
```
6 classifications + deciding factors.
```

> [!hint]- Hint
> Spiky/event-driven/short = great. Sustained/stateful/long = poor.

> [!success]- Solution
> - (a) **Great** — the canonical use case: event-driven, short, spiky.
> - (b) **Poor** — sustained high throughput: per-invocation cost crosses over container cost; cold starts hurt p99. Containers win.
> - (c) **Great** — runs rarely, scale-to-zero between runs (watch the 15-min limit; else Step Functions/Batch).
> - (d) **Poor** — long-lived stateful connections fight the FaaS model (API Gateway WebSockets exists but adds complexity; a container is simpler).
> - (e) **Great** — tiny, spiky, event-shaped.
> - (f) **Poor** — exceeds execution limits → AWS Batch/ECS task or MediaConvert.

### Problem 10: Distributed monolith diagnosis

A team of 8 runs 22 microservices. Symptoms: releases are coordinated "release trains" of 10+ services; a shared `common-models` library version-bumps force mass redeploys; most requests fan out through 5+ synchronous hops; one Postgres serves 9 services. Diagnose, name each smell, and prescribe a 3-step recovery.

**Expected output:**
```
Diagnosis + 4 named smells + 3-step plan.
```

> [!hint]- Hint
> The cure is usually *fewer* services, not better tooling.

> [!success]- Solution
> **Diagnosis: distributed monolith** — microservices cost without microservices benefit.
> Smells: (1) coordinated deploys = no independent deployability (the defining trait, absent); (2) shared model library = compile-time coupling across services; (3) deep sync chains = temporal coupling + multiplied latency/failure ([[02 - Quality Attributes Drills|availability math]]); (4) shared DB across 9 services = no data ownership.
> **Recovery:** (1) merge chatty, transaction-sharing services along domain seams — target ~5–6 services for 8 devs (service-based); (2) kill `common-models`: each service owns its contracts, share only via versioned API/event schemas; (3) assign data ownership — each surviving service gets its schema, others access via API, enforced by revoking cross-schema DB grants.
