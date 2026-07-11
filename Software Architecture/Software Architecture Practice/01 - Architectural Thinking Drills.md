---
title: "01 - Architectural Thinking Drills"
date: 2026-07-11
tags:
  - software-architecture
  - architectural-thinking
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 01 — Architectural Thinking Drills

> [!abstract] What you're drilling
> Definitions, architecture-vs-design judgment, breadth-vs-depth, component identification. Answer **in writing, from scratch** before opening solutions. Each problem adds one concept.

---

### Problem 1: Define it in your own words

Without looking at notes, write the 4-part definition of software architecture and give one concrete example for each part, using a ride-hailing app (like Grab) as the system.

**Expected output:**
```
Structure, characteristics, decisions, design principles — each with a Grab-specific example.
```

> [!hint]- Hint
> Structure = style; characteristics = -ilities; decisions = rules; principles = guidelines.

> [!success]- Solution
> - **Structure:** microservices (Ride, Pricing, Driver, Payment services) with event-driven dispatch.
> - **Characteristics:** availability (rides 24/7), low latency (matching < 2s), elasticity (rain spikes).
> - **Decision:** "Driver location updates must go through the location stream, never direct DB writes."
> - **Design principle:** "Prefer async events between services; sync calls only for user-facing reads."

### Problem 2: Architecture or design?

Classify each as architecture (hard to change) or design (cheap to change), with one sentence why: (a) choosing PostgreSQL vs DynamoDB, (b) renaming `OrderMgr` to `OrderService`, (c) sync REST vs async events between two services, (d) strategy pattern vs if/else inside pricing, (e) monorepo vs polyrepo, (f) which JSON field naming convention.

**Expected output:**
```
a, c, e = architecture; b, d, f = design — each with reasoning about cost-to-change.
```

> [!hint]- Hint
> Ask: "If wrong, how expensive is the fix?"

> [!success]- Solution
> - (a) **Architecture** — data model, queries, and consistency assumptions get baked in everywhere.
> - (b) **Design** — one refactor commit.
> - (c) **Architecture** — changes failure modes, consistency, and both services' code.
> - (d) **Design** — internal to one component.
> - (e) **Architecture(-ish)** — affects CI/CD, versioning, team workflow across the org; expensive to reverse.
> - (f) **Design** — annoying but mechanical to change (though it becomes architectural once external consumers depend on it — note the spectrum!).

### Problem 3: Breadth over depth

Your team must pick a message broker. You know RabbitMQ deeply, nothing else. List the minimum *breadth* knowledge you need about Kafka and SQS to make a fair choice (5 dimensions), without becoming an expert in either.

**Expected output:**
```
5 comparison dimensions + one-line answer per broker per dimension.
```

> [!hint]- Hint
> Think delivery semantics, ordering, retention/replay, ops burden, throughput/cost.

> [!success]- Solution
> | Dimension | RabbitMQ | Kafka | SQS |
> |---|---|---|---|
> | Model | smart broker, routing | distributed log, dumb broker | managed queue |
> | Ordering | per-queue-ish | per-partition guaranteed | FIFO queues only (limited TPS) |
> | Replay | no (consumed = gone) | yes (retention window) | no |
> | Ops | self-host or managed | heavy (or MSK/Confluent $) | zero ops |
> | Sweet spot | task queues, RPC-ish, routing | streams, replay, high throughput | simple decoupling on AWS |
>
> The point: an architect needs *this table*, not three certifications.

### Problem 4: Extract business drivers

The CEO says: "We're expanding to 3 new countries next quarter, regulators require local data storage, and we can't afford more downtime like last month's." Extract the architecture characteristics (with target metrics you'd propose).

**Expected output:**
```
3+ characteristics, each traced to a CEO sentence, each measurable.
```

> [!hint]- Hint
> One sentence = one or more -ilities.

> [!success]- Solution
> - "3 new countries next quarter" → **extensibility/deployability** (new-country rollout ≤ 2 weeks config, not code fork) and **i18n readiness**.
> - "local data storage" → **compliance/data residency** (user PII stored in-region; verified by audit).
> - "downtime like last month" → **availability** (propose 99.9% monthly = ~43 min budget) + **recoverability** (RTO 15 min, RPO 5 min).

### Problem 5: Identify components

For a hospital appointment system (patients book slots, doctors set availability, SMS reminders, billing to insurers, admin reports), list initial components and assign each requirement to one.

**Expected output:**
```
5–7 components, each with responsibilities; no God components.
```

> [!hint]- Hint
> Nouns in the requirements are candidate components; verbs are their responsibilities.

> [!success]- Solution
> - **Scheduling** — slot search, booking, cancellation (patient + doctor availability)
> - **PatientRecords** — demographics, contact prefs
> - **Notification** — SMS/email reminders (consumes booking events)
> - **Billing** — insurer claims, invoicing
> - **Reporting** — admin utilization reports (read-only, batch OK)
> - **IdentityAccess** — login, roles (patient/doctor/admin)
>
> Note Scheduling ≠ PatientRecords: different change rates and privacy profiles.

### Problem 6: Spot the God component

In Problem 5's system, a teammate proposes one `AppointmentManager` that books slots, sends SMS, creates invoices, and generates reports. Give 3 architecture-level arguments against it, referencing characteristics.

**Expected output:**
```
3 arguments tied to -ilities, not taste.
```

> [!hint]- Hint
> Different responsibilities have different scalability, availability, and change-rate needs.

> [!success]- Solution
> 1. **Differing operational needs:** reminders are async/batchy and can lag; booking is interactive and must be fast — one component forces one scaling profile on both.
> 2. **Fault isolation:** an SMS-provider outage shouldn't block bookings; inside one component it can.
> 3. **Maintainability/testability:** billing rules change on insurer schedules, booking UX changes weekly — coupled code means every change risks everything, and tests must stand up the world.

### Problem 7: Technical vs domain partitioning

Draw (ASCII) the same e-commerce app partitioned both ways. Then answer: a request arrives — "add gift-wrapping option to checkout." Which partitioning touches fewer top-level units, and why does that matter for team ownership?

**Expected output:**
```
Two ASCII structures + reasoning about locality of change.
```

> [!hint]- Hint
> Technical = layers; domain = Orders/Catalog/Customers.

> [!success]- Solution
> ```
> TECHNICAL                     DOMAIN
> ├── presentation/             ├── catalog/
> ├── business/                 ├── checkout/     ◄ gift-wrap lands here only
> ├── persistence/              ├── payments/
> └── database                  └── customers/
> ```
> Gift-wrap in technical partitioning touches presentation + business + persistence (3 units, possibly 3 owners). In domain partitioning it lands inside `checkout/` (1 unit, 1 team). Business changes arrive per-domain, so domain partitioning localizes them — this is also why microservices split by domain, never by layer.

### Problem 8: The hands-on architect

You're the architect and the sprint is behind. The tech lead asks you to own the payment-gateway integration (critical path, 3 weeks). Name the trap, then list 4 alternative ways to stay hands-on without taking critical-path work.

**Expected output:**
```
The bottleneck trap + 4 alternatives.
```

> [!hint]- Hint
> Architects get pulled into meetings; critical-path work then blocks the team.

> [!success]- Solution
> **Trap:** the architect becomes the delivery bottleneck — meetings and reviews mean your critical-path code is always late, blocking everyone.
> Alternatives: (1) build the *proof-of-concept* / walking skeleton for the risky part, then hand it over; (2) take technical-debt items or bug fixes off the backlog (non-blocking); (3) pair/mob with the dev who owns the integration; (4) write the fitness functions / contract tests that verify the integration's architectural rules.

### Problem 9: First-law reflex

A senior dev says: "GraphQL is simply better than REST — we should use it everywhere." Apply the First Law of Software Architecture: produce the trade-off table they skipped.

**Expected output:**
```
A GraphQL-vs-REST trade-off table (4+ rows) + a context where each wins.
```

> [!hint]- Hint
> "Simply better" = trade-off not yet identified.

> [!success]- Solution
> | | GraphQL | REST |
> |---|---|---|
> | Client flexibility | ✅ ask for exactly what you need | ❌ over/under-fetching |
> | Caching | ❌ hard (POST, per-query) | ✅ HTTP caching, CDNs |
> | Server complexity | ❌ resolvers, N+1, query cost limits | ✅ simple, mature tooling |
> | Multiple diverse clients | ✅ shines (mobile+web+partners) | needs BFF per client type |
> | Observability/rate-limiting | ❌ one endpoint hides everything | ✅ per-route metrics/limits |
>
> GraphQL wins: many client types with divergent data needs (e.g., mobile app + web + partner portal). REST wins: public API with heavy caching, simple resource CRUD, small team.

### Problem 10: Run the full thinking loop

Timed drill (30 min, from memory): a startup wants a "Duolingo for cooking" app — video lessons, daily streaks, push notifications, subscription payments, 2 devs, launch in 3 months. Produce: top-3 characteristics (measurable), component list, structure choice, and 2 recorded trade-offs.

**Expected output:**
```
Characteristics table, component list, one-paragraph structure decision, 2 trade-off statements.
```

> [!hint]- Hint
> 2 devs + 3 months is itself the loudest requirement.

> [!success]- Solution
> **Characteristics:** simplicity/time-to-market (MVP live in 12 weeks), cost (< $200/mo infra), availability 99.5% (lessons cached client-side soften outages).
> **Components:** Lessons, Progress/Streaks, Notifications, Subscription/Billing, Identity.
> **Structure:** modular monolith (domain modules), Postgres, hosted on 1–2 VMs or a PaaS; video via a managed service (Mux/Cloudflare Stream) — never build video infra yourself; push via FCM/APNs through a small async worker; payments via Stripe/RevenueCat.
> **Trade-offs recorded:** (1) Monolith trades independent scaling for speed/simplicity — acceptable: scale ceiling is far above 3-month horizon; revisit at ~50k DAU. (2) Managed video trades cost-per-view + lock-in for months of saved eng time — acceptable: video infra is not our differentiator.
