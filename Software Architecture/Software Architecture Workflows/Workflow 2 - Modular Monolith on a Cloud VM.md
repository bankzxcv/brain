---
title: "Workflow 2 - Modular Monolith on a Cloud VM"
date: 2026-07-11
tags:
  - software-architecture
  - modular-monolith
  - cloud
  - vm
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 2 — Modular Monolith on Cloud VMs (3-Tier)

> [!abstract] Project
> Build and deploy **"Sarn"**, a marketplace for local craftspeople (listings, orders, payments, reviews), as a domain-partitioned modular monolith on cloud VMs — the honest workhorse architecture. You'll structure the code, enforce module boundaries, then deploy it as a proper 3-tier, multi-AZ setup and make it survive failure. Local practice: everything runs with docker-compose before touching the cloud; VM steps map 1:1 to AWS/GCP.

**Architecture:**

```
                      Internet
                         │
              ┌──────────▼──────────┐
              │  Load balancer      │  (ALB / Cloud LB, TLS terminates here)
              │  across 2 AZs       │
              └─────┬───────┬───────┘
        AZ-a  ┌─────▼───┐ ┌─▼───────┐  AZ-b        ── app tier
              │ VM: app │ │ VM: app │  (stateless! sessions in Redis)
              └───┬───┬─┘ └─┬───┬───┘
                  │   └─────┼───┼────────┐
              ┌───▼─────────▼┐ ┌▼────────▼─┐        ── data tier
              │ Postgres     │ │ Redis      │  (private subnets)
              │ primary AZ-a │ │            │
              │ standby AZ-b │ └────────────┘
              └──────────────┘
   S3 + CDN: product images, static assets
   Worker VM (or same VMs): queue consumers (emails, image resize)
```

---

## Step 1 — Module structure with enforced boundaries

**Do:** lay out the Go (or your language) project: modules `catalog`, `orders`, `payments`, `reviews`, `identity`; each with a public facade and private internals; per-module tables. Write the two boundary rules you'll enforce in CI.

> [!success]- Solution
> ```
> sarn/
>   cmd/api/main.go            (wires everything; the ONLY place modules meet)
>   modules/
>     catalog/   catalog.go    ◄ public facade (interface + constructor)
>                internal/     ◄ handlers, service, repo, unimportable outside
>     orders/    orders.go  internal/...
>     payments/  payments.go internal/...
>     reviews/   reviews.go  internal/...
>     identity/  identity.go internal/...
>   migrations/  0001_catalog_*.sql, 0002_orders_*.sql ...
>   platform/    db, queue, config, logging (shared plumbing only — no domain!)
> ```
> Facade example: `orders.Service` interface { PlaceOrder, GetOrder, ListByBuyer }; `orders.New(db, catalog catalog.Service, pay payments.Service)` — cross-module calls are **interface calls through constructors**, so dependencies are explicit in `main.go`.
> **CI rules:** (1) Go's `internal/` gives rule one for free; add `depguard`: `modules/X` may import `modules/Y` (facade) but never `modules/Y/internal`. (2) Migration lint: files prefixed per module; a script fails CI if module code contains another module's table prefix in SQL.

## Step 2 — Wire one real cross-module flow

**Do:** implement `PlaceOrder`: validate items via catalog facade, create order, authorize payment via payments facade (which wraps a PSP client), all in one DB transaction where appropriate + outbox event `order.placed`.

> [!success]- Solution
> ```go
> func (s *service) PlaceOrder(ctx context.Context, cmd PlaceOrderCmd) (OrderID, error) {
>     items, err := s.catalog.ValidateItems(ctx, cmd.Items) // cross-module, in-process
>     if err != nil { return 0, fmt.Errorf("invalid items: %w", err) }
>
>     auth, err := s.payments.Authorize(ctx, cmd.BuyerID, total(items)) // external PSP inside
>     if err != nil { return 0, ErrPaymentDeclined }
>
>     var id OrderID
>     err = s.tx(ctx, func(q Queries) error {          // ONE transaction:
>         id, err = q.InsertOrder(cmd, items, auth.Ref) // order rows
>         if err != nil { return err }
>         return q.InsertOutbox("order.placed", OrderPlaced{ID: id}) // + outbox
>     })
>     return id, err
> }
> ```
> Notes: payment authorization happens *before* the tx (external calls never inside DB transactions — connection pinning + long locks); on tx failure, a reversal job voids the auth (mini-saga: [[05 - Communication and Data Drills#Problem 4 Design the saga|drill]]). The monolith's gift: `ValidateItems` is a function call — no network fallacies ([[04 - Distributed Styles Drills#Problem 2 Fallacy autopsy|drill]]).

## Step 3 — Make the app tier stateless

**Do:** audit for statefulness: sessions, uploaded images, in-memory caches, cron jobs. Fix each so any VM can serve any request and VMs can die freely.

> [!success]- Solution
> - **Sessions** → Redis-backed (or short-lived JWT + refresh); kill in-memory session maps.
> - **Uploads** → stream to S3/GCS, store keys; never local disk. Serve via CDN.
> - **Caches** → in-memory caches are fine *if* they're pure read-through with TTL (each VM warms independently); anything requiring coherence → Redis.
> - **Cron** → moves to one of: leader election (Redis lock), the queue (scheduled messages), or a dedicated small worker — otherwise 2 VMs = every job runs twice.
> Statelessness is the price of horizontal scaling, rolling deploys, and "instances are cattle" — pay it in full.

## Step 4 — Provision the 3-tier layout (local → cloud)

**Do:** first replicate the topology in docker-compose (nginx LB + 2 app containers + postgres + redis). Then write the cloud layout: VPC with public/private subnets across 2 AZs, security groups per tier, ALB, 2 app VMs (or an autoscaling group), managed Postgres multi-AZ, managed Redis. List the exact security-group rules.

> [!success]- Solution
> Compose: `nginx (80) → app1, app2 (8080) → postgres (5432), redis (6379)` — experiencing round-robin + a killed container locally teaches 80% of the cloud lesson free.
> Cloud (AWS vocabulary; GCP maps directly):
> - **VPC** 10.0.0.0/16: public subnets 10.0.1–2.0/24 (AZ a/b) for ALB + NAT; private app subnets 10.0.11–12.0/24; private data subnets 10.0.21–22.0/24.
> - **Security groups (least privilege, SG-references not CIDRs):** `sg-alb`: in 443 from 0.0.0.0/0 · `sg-app`: in 8080 from sg-alb only · `sg-db`: in 5432 from sg-app only · `sg-redis`: in 6379 from sg-app only. Egress app → 443 (PSP, S3 gateway endpoint).
> - **RDS Postgres multi-AZ** (standby in AZ-b, automatic failover ~60–120s), **ElastiCache Redis**, **S3 + CloudFront**, app VMs in an **auto-scaling group** (min 2, one per AZ) behind the ALB with health check `GET /healthz` (checks DB + Redis connectivity, returns build SHA).

## Step 5 — Deploy pipeline & zero-downtime releases

**Do:** design the release: image/AMI build, rolling deploy through the ASG, DB migrations strategy (the classic trap), and rollback story.

> [!success]- Solution
> Pipeline: CI runs tests + arch rules (Step 1) → builds artifact (container image even on VMs — run via systemd+docker; identical local/cloud) → staging → prod rolling: ALB drains one instance (connection draining 30s), replace, health-check gate, next.
> **Migrations — expand/contract, always:** deploying code and schema together across a rolling fleet means old code briefly runs on new schema. So: (1) *expand* migration first (add nullable column/table — old code ignores it), (2) deploy code that writes both / reads new, (3) backfill, (4) *contract* later (drop old column) once no code references it. `DROP COLUMN` in the same release as the code change is the classic outage.
> **Rollback:** previous image redeploy (fast, safe *because* schema is always backward-compatible one version). Canary option: ALB weighted target groups 5% → 100%.

## Step 6 — Failure drills (the payoff of multi-AZ)

**Do:** predict system behavior, then verify each: (a) kill one app VM during traffic, (b) fail over the DB primary, (c) Redis dies, (d) the PSP times out, (e) AZ-a goes dark.

> [!success]- Solution
> - (a) ALB health check fails in ~2×interval → traffic shifts to the other VM; ASG replaces the dead one. Users: nothing (maybe a blip of in-flight requests). If users noticed → sessions weren't in Redis (Step 3 failed).
> - (b) RDS failover: ~60–120s of write errors. App must: fail fast, retry with backoff, show a friendly error — and *recover connections* (connection pool must re-resolve DNS! stale pools are the classic post-failover outage).
> - (c) Sessions gone (users re-login) unless Redis has replica/AOF; caches cold → DB load spike → this is why you load-test with cold cache. Decide: is Redis a hard or soft dependency? Make cache-misses degrade, not 500.
> - (d) Timeout (1–2s) + circuit breaker → orders queue as "payment pending" or fail gracefully with retry UX. Never hang threads on the PSP ([[02 - Quality Attributes Drills#Problem 3 Availability math|availability math]]).
> - (e) ALB routes to AZ-b VM; RDS fails over to AZ-b standby; Redis (if single-AZ — decide!) may die → see (c). Capacity halves → ASG launches replacements in AZ-b. Survives, degraded. That's the design working.

## Step 7 — Observability & cost check

**Do:** define the minimum viable observability stack and estimate monthly cost of the whole setup.

> [!success]- Solution
> **Observability:** structured JSON logs shipped centrally (CloudWatch/Loki); RED metrics per module (requests, errors, duration — labeled by module, the modular monolith's namespaces pay off here); traces optional at this scale but add request IDs everywhere now; alerts: p99 latency, 5xx rate, DB CPU, queue depth, disk. Dashboard per module + one golden overview.
> **Cost (~ap-southeast-1 ballpark):** 2× t3.medium ~$60 · ALB ~$25 · RDS t4g.medium multi-AZ ~$120 · ElastiCache t4g.small ~$25 · S3+CDN ~$20 · NAT ~$35 · misc ~$15 ≈ **~$300/mo**. Compare with [[06 - Trade-off Analysis Drills#Problem 5 Cost model duel|the cost duel drill]] — and notice again the DB is the floor.

---

## Extensions

1. Convert the VM deployment to ECS Fargate — what disappears (AMIs, patching), what appears (task defs), what stays identical (everything architectural)?
2. Add a read replica and route reporting queries to it — congratulations, you've built [[Software Architecture Study#9.3 CQRS — Command Query Responsibility Segregation|CQRS level 2]].
3. Extract `payments` into a service *without changing any other module's code* — if Step 1 was done right, only `main.go` wiring changes to an HTTP client implementing the same facade interface. Time yourself.
4. Write the ADR for staying on VMs vs moving to Kubernetes, with revisit triggers.
