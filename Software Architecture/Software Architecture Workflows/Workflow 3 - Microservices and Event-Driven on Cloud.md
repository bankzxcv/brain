---
title: "Workflow 3 - Microservices and Event-Driven on Cloud"
date: 2026-07-11
tags:
  - software-architecture
  - microservices
  - event-driven
  - kubernetes
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 3 — Microservices & Event-Driven on Cloud (Kubernetes)

> [!abstract] Project
> Evolve **"Sarn"** (the marketplace from [[Workflow 2 - Modular Monolith on a Cloud VM]]) at a later stage: 40 engineers, 3 countries, the monolith's deploy queue is the bottleneck. You'll do the extraction properly: choose seams, split data, build the event backbone, run it on Kubernetes, and handle the distributed-systems tax honestly. Local practice: minikube + a local Kafka/RabbitMQ before any cloud.

**Architecture (target):**

```
                          ┌─────────────┐
     clients ───────────► │ API Gateway │ (authn, rate limits, routing)
                          └─┬────┬────┬─┘
                ┌───────────┘    │    └────────────┐
          ┌─────▼─────┐   ┌─────▼─────┐   ┌────────▼──┐
          │ Catalog   │   │ Orders    │   │ Payments  │      each:
          │ svc + DB  │   │ svc + DB  │   │ svc + DB  │      own repo, own
          └─────┬─────┘   └──┬─────┬──┘   └────┬──────┘      pipeline, own DB,
                │            │outbox│          │             own team
                │        ┌───▼─────▼───────────▼───┐
                └───────►│  Event backbone (Kafka)  │ topics: order.events,
                         └───┬──────┬──────┬────────┘        payment.events...
                     ┌───────▼┐ ┌───▼────┐ ┌▼──────────┐
                     │Notifier│ │Search  │ │Settlement │  (consumers)
                     └────────┘ │indexer │ └───────────┘
                                └────────┘
     Platform layer: Kubernetes (EKS/GKE), CI/CD per service, central logging,
     tracing (OTel), metrics, secrets, service discovery (K8s DNS)
```

---

## Step 1 — Decide WHAT to extract (and what not)

**Do:** the monolith has modules catalog, orders, payments, reviews, identity + the pains: payments team blocked by deploy queue and needs PCI-scope isolation; catalog search is slow and read-heavy; reviews barely change. Apply granularity drivers ([[04 - Distributed Styles Drills#Problem 5 Granularity referee|drill]]) and produce the extraction order.

> [!success]- Solution
> | Module | Drivers to split | Verdict |
> |---|---|---|
> | Payments | security/compliance (PCI scope shrinks to one service), deploy autonomy, team ownership | **Extract 1st** |
> | Catalog | scale (read-heavy, needs search index), volatility | **Extract 2nd** (with its own read model) |
> | Orders | transaction core, moderate | Extract 3rd, *after* the event backbone exists (it produces the events everyone wants) |
> | Reviews | none — stable, low traffic | **Stays** in the monolith (which quietly becomes "just another service") |
> | Identity | shared by all — but generic | Replace with managed auth (Cognito/Auth0) rather than extract |
>
> The end state is 4–5 services + a residual monolith — **not** 25 services. Strangler-fig, not big-bang rewrite.

## Step 2 — Split the data (the hard part)

**Do:** payments tables live in the shared Postgres and orders JOIN against them. Design the data extraction: new ownership, how orders gets payment status now, and the migration sequence with zero downtime.

> [!success]- Solution
> **Ownership:** payments gets its own DB; the `payment_authorizations`, `captures`, `refunds` tables move. Orders may **no longer JOIN** — it gets payment status via: (a) sync API for on-demand queries, and (b) subscribing to `payment.events` to keep a small local copy (`order_payment_status`: order_id, status, updated_at) — a *replicated read model*, the standard cure for lost JOINs.
> **Migration sequence (expand/contract at system level):**
> 1. Payments service deployed, backed by *the same tables* (monolith stops writing them — facade already routed calls in [[Workflow 2 - Modular Monolith on a Cloud VM|W2 Step 1]]).
> 2. Dual-read verification period (shadow traffic, compare).
> 3. New DB stood up; logical replication copies tables; cutover in a maintenance window (or CDC-based near-zero-downtime cutover).
> 4. Cross-schema grants revoked; the JOINs are dead for real. Fitness function: monolith's DB user loses SELECT on payment tables — violations now fail loudly in staging, not silently in prod.

## Step 3 — Design the event backbone

**Do:** choose Kafka vs RabbitMQ vs SNS/SQS for this context; define topics, event schema strategy, and the delivery guarantees consumers must handle.

> [!success]- Solution
> **Choice: Kafka (MSK)** — deciding factors: multiple consumers per event (notifier, search, settlement, analytics all read `order.events`), replay needed to (re)build read models (Step 2's replicated status table, search index), partition-ordering per order ID. (SNS/SQS would win at smaller scale/ops budget; RabbitMQ if routing-heavy work-queues dominated. Justify yours — see [[01 - Architectural Thinking Drills#Problem 3 Breadth over depth|breadth drill]].)
> **Topics:** `order.events`, `payment.events`, `catalog.events` — partition key = aggregate ID (all events for order 42 stay ordered). **Schemas:** Avro/Protobuf + schema registry, *backward-compatible evolution only* (add optional fields; never rename/retype — the schema is a public API frozen in time).
> **Consumer contract:** at-least-once + possible reordering *across* partitions → every consumer idempotent ([[04 - Distributed Styles Drills#Problem 7 Idempotent consumer|drill]]), every producer uses the outbox ([[05 - Communication and Data Drills#Problem 9 The outbox problem|drill]]), every topic gets a DLQ + alert.

## Step 4 — Kubernetes deployment (local first)

**Do:** on minikube: deploy 2 services + Kafka (Strimzi or docker-compose Kafka alongside), with Deployments, Services, HPA, probes, and resource limits. List the manifest essentials per service and what each probe must actually check.

> [!success]- Solution
> Per service: `Deployment` (replicas ≥ 2, `PodDisruptionBudget`, resources requests/limits — no limits = one leaky service evicts neighbors), `Service` (ClusterIP; K8s DNS is your service discovery), `HPA` (CPU or better: RPS/lag-based), `ConfigMap`+`Secret`, probes:
> - **liveness** = "process wedged? restart me" — check *only self* (never DB! or a DB blip restarts your whole fleet in a loop);
> - **readiness** = "can I serve? else remove from LB" — this one checks critical deps;
> - **startup** for slow boots.
> Ingress/Gateway API at the edge (authn via gateway + JWT propagation). Cloud = same manifests on EKS/GKE + managed node groups, cluster-autoscaler, IRSA/Workload Identity for AWS/GCP creds. The local→cloud identity of manifests is the point of Kubernetes.

## Step 5 — The checkout saga across services

**Do:** implement checkout spanning orders → payments → catalog(stock): choose orchestration or choreography ([[04 - Distributed Styles Drills#Problem 6 Broker vs mediator EDA|drill]]), define the flow including failures, and decide what the *user* sees during the seconds it runs.

> [!success]- Solution
> **Orchestrated** (sequential + compensations = orchestration's home turf): Orders service hosts the saga (or Temporal if adopted): `PlaceOrder` persists order(PENDING)+outbox → saga: reserve stock → authorize payment → confirm order. Compensations: release stock; void auth. Every step idempotent + retried with backoff; saga state persisted (survives pod death — a K8s certainty, not a possibility).
> **UX for eventual consistency:** API returns `202 + orderID` instantly; client subscribes (SSE/websocket/poll) to order status; UI shows "confirming your order…" honestly. Checkout p99 *user-perceived* = the saga's length — so budget it: stock 50ms + payment PSP 800ms + confirm 20ms ≈ acceptable. If the PSP is slow, the *user* waits — a trade-off no architecture removes, only relocates.

## Step 6 — Observability or death

**Do:** a customer reports "order stuck." Trace the debugging path across 4 services and list the observability stack that makes it a 5-minute investigation instead of a war room.

> [!success]- Solution
> Stack: **OpenTelemetry** everywhere — trace context propagated via HTTP headers *and Kafka message headers* (the one everyone forgets); traces → Tempo/Jaeger; RED metrics + consumer-lag per service → Prometheus/Grafana; structured logs with `trace_id` + `order_id` → Loki/CloudWatch, cross-linked from traces.
> The 5-minute path: search logs by order_id → find trace_id → open trace → see saga stopped at `payment.authorize` retrying → payments dashboard shows PSP 5xx spike → DLQ has the parked message → replay after PSP recovers. Every hop in that sentence is a tool you must have *before* the incident. Alerts that matter: consumer lag growth, DLQ depth > 0, saga age > SLA, per-service error budgets.

## Step 7 — The honest retrospective

**Do:** write the costs-incurred list (what got worse vs the monolith) and the benefits-realized list, and the one-paragraph "when should a team NOT do this."

> [!success]- Solution
> **Worse:** infra bill ~5–8x (cluster + Kafka + observability + NAT...); every workflow crosses networks (fallacies now daily reality); debugging requires tooling and discipline; local dev needs orchestration; schema/contract governance is a standing meeting; eventual consistency leaks into UX and support tickets.
> **Better:** payments deploys 20x/week untouched by others; PCI audit scope = one service; catalog scales its read path independently; a bad deploy takes down one capability, not the company; teams own services end-to-end (Conway alignment).
> **Don't do this if:** you have < ~15–20 engineers, one team, no platform capacity, or your pain is code quality rather than *organizational scaling* — microservices solve a people-coordination problem and charge a distributed-systems price; teams that buy them for performance or fashion get [[04 - Distributed Styles Drills#Problem 10 Distributed monolith diagnosis|the distributed monolith]].

---

## Extensions

1. Add a new consumer (fraud-scoring) fed from `order.events` + `payment.events` without touching any producer — feel EDA's extension superpower.
2. Replace the orchestrated saga with Temporal; compare code volume and visibility.
3. Kill a Kafka broker in minikube mid-saga; verify nothing is lost (acks=all, min.insync.replicas=2) and consumers resume.
4. Do the [[06 - Trade-off Analysis Drills#Problem 1 Build a weighted matrix|weighted matrix]] for *your* current workplace: monolith vs service-based vs this. Defend it in one page.
