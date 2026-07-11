---
title: "06 - Trade-off Analysis Drills"
date: 2026-07-11
tags:
  - software-architecture
  - trade-offs
  - solution-architecture
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 06 — Solution-Architect Trade-off Drills

> [!abstract] What you're drilling
> The capstone skill: weighted trade-off matrices, ADR writing, sensitivity analysis, cost modeling, and defending decisions to stakeholders. See [[Software Architecture Study#12. Solution-Architect Trade-off Analysis|Study §12]]. These drills are interview-grade.

---

### Problem 1: Build a weighted matrix

A fintech startup (8 devs, Series A, must pass SOC 2 within a year) is choosing between: modular monolith on ECS, service-based on ECS, microservices on EKS. Business priorities: ship weekly (critical), pass audit (critical), handle 10x growth in 18 months (important), minimize burn (important). Build the weighted matrix and declare a winner.

**Expected output:**
```
Criteria with weights, scores 1–5, totals, winner + one-sentence rationale.
```

> [!hint]- Hint
> Audit favors fewer moving parts; weekly shipping favors deploy simplicity at this team size.

> [!success]- Solution
> | Criterion (w) | Mod. monolith | Service-based | Microservices |
> |---|---|---|---|
> | Delivery speed (×3) | 5→15 | 4→12 | 2→6 |
> | Auditability/SOC2 (×3) | 4→12 | 4→12 | 2→6 |
> | 10x growth (×2) | 3→6 | 4→8 | 5→10 |
> | Burn/run cost (×2) | 5→10 | 4→8 | 2→4 |
> | Team fit 8 devs (×2) | 5→10 | 4→8 | 2→4 |
> | **Total** | **53** | **48** | **30** |
>
> **Winner: modular monolith** — 10x growth is the only criterion it loses, and 10x on a stateless monolith is a load-balancer problem, not an architecture problem. SOC 2 audits go faster with one deployable, one DB, one audit boundary. Microservices lose on every weighted-critical axis for a team of 8.

### Problem 2: Sensitivity analysis

Take Problem 1's matrix. For each of these events, state whether the decision flips: (a) growth projection revised to 100x, (b) team doubles to 16 in 6 months, (c) a second product line must reuse payments, (d) run-cost weight drops (new funding). Which single assumption is the decision most sensitive to?

**Expected output:**
```
4 flip-analyses + the most sensitive assumption.
```

> [!hint]- Hint
> Re-score only the affected cells; check if ranking changes.

> [!success]- Solution
> - (a) 100x: scalability weight ×2→×3 and monolith score 3→2 → totals ~monolith 50, service-based 51: **flips to service-based** (not to microservices — audit still dominates).
> - (b) 16 devs: team-fit gap narrows (micro 2→3), but monolith still leads: **no flip** yet; at 25+ devs it would.
> - (c) Payments reuse: introduces a *new criterion* (reusability ×2: mono 2, service 5, micro 5) → service-based ≈ monolith-with-extracted-payments: **partial flip** — extract exactly one service.
> - (d) Cost weight ×2→×1: totals drop uniformly-ish: **no flip**.
> **Most sensitive to team size + growth combined** — so the ADR must state: "revisit when headcount > 15 or sustained load > Nx" with the modular seams as the pre-built exit.

### Problem 3: Write the ADR

Write the full ADR for Problem 1's decision (context, decision, consequences, alternatives), including the revisit triggers from Problem 2. Max 250 words — brevity is part of the drill.

**Expected output:**
```
A complete ADR in standard format, ≤250 words.
```

> [!hint]- Hint
> Consequences must include the *negative* ones.

> [!success]- Solution
> ```markdown
> # ADR-001: Modular monolith on ECS
> Status: Accepted (2026-07-11)
>
> ## Context
> 8 engineers, weekly release target, SOC 2 audit within 12 months,
> projected 10x traffic growth in 18 months, Series A burn constraints.
> Candidates: modular monolith, service-based (5 services), microservices.
> Weighted matrix: 53 / 48 / 30 (see analysis doc).
>
> ## Decision
> Single deployable Go application, domain-partitioned modules
> (identity, ledger, payments, reporting) with enforced boundaries
> (internal packages, per-module schemas, arch tests in CI).
> Deployed on ECS Fargate behind ALB, RDS Postgres multi-AZ.
>
> ## Consequences
> + Weekly deploys with one pipeline; one audit boundary for SOC 2
> + ~$1.4k/mo infra vs ~$5k+ for EKS platform
> + Module seams allow future extraction without rewrite
> - Whole-app blast radius per deploy (mitigate: canary + auto-rollback)
> - Scaling is all-or-nothing (acceptable: stateless, LB-scalable to ~10x)
> - Payments cannot be reused by other products without extraction
>
> ## Revisit triggers
> Headcount > 15; sustained load > 10x baseline; second product
> needs payments → extract payments module first.
>
> ## Alternatives
> Service-based: viable runner-up, deferred until a trigger fires.
> Microservices: rejected — platform cost and audit surface
> unjustifiable at current team size.
> ```

### Problem 4: Buy vs build vs managed

Your product needs full-text search. Options: (A) Postgres `tsvector` (already have Postgres), (B) self-hosted Elasticsearch on EC2, (C) managed OpenSearch/Algolia. Requirements: 2M documents, 50 searches/sec peak, typo tolerance, faceting, 1 infra engineer total. Analyze and decide.

**Expected output:**
```
Comparison across capability, ops, cost + decision.
```

> [!hint]- Hint
> The scarcest resource in the requirements is not compute.

> [!success]- Solution
> | | Postgres FTS | Self-hosted ES | Managed (OpenSearch/Algolia) |
> |---|---|---|---|
> | Typo tolerance/facets | weak/manual | ✅ | ✅ |
> | Ops burden | ~zero (existing DB) | cluster care: JVM, shards, upgrades | low |
> | Cost | $0 marginal | EC2 + the engineer's *time* | $$ (Algolia per-op can sting) |
> | Scale to reqs | 2M docs OK; facets painful | ✅ | ✅ |
>
> **Decision: C (managed).** The binding constraint is *1 infra engineer* — self-hosted ES quietly consumes 20–30% of them. Postgres FTS fails the typo-tolerance/faceting requirements (would be the pick if requirements were plain keyword search — always check if the boring option meets the real bar). Between OpenSearch and Algolia: OpenSearch if cost-sensitive at volume, Algolia if dev-speed matters most. Record the lock-in trade-off in an ADR.

### Problem 5: Cost model duel

Estimate a rough monthly cost for the same workload (steady 200 RPS API + Postgres, ap-southeast-1-ish prices) as: (A) 3× t3.large VMs + ALB + RDS db.r6g.large multi-AZ; (B) ECS Fargate (4 tasks, 1vCPU/2GB) + same RDS; (C) Lambda (200 RPS × 120ms × 512MB) + API Gateway + same RDS. Ballpark numbers are fine — the drill is the *method* and the crossover conclusion.

**Expected output:**
```
Three cost breakdowns + when each wins.
```

> [!hint]- Hint
> Lambda pricing: requests + GB-seconds. 200 RPS is ~518M requests/month.

> [!success]- Solution
> Ballpark (prices drift — method over digits):
> - **(A) VMs:** 3× t3.large ≈ $180 + ALB ≈ $25 + RDS r6g.large multi-AZ ≈ $470 ≈ **~$675/mo**
> - **(B) Fargate:** 4× (1vCPU+2GB) ≈ $145 + ALB + RDS ≈ **~$640/mo**
> - **(C) Lambda:** 518M req ≈ $104 + GB-s: 518M × 0.12s × 0.5GB ≈ 31M GB-s ≈ $517 + API GW ≈ $520+ + RDS ≈ **~$1,600+/mo** (plus RDS Proxy for connection storms)
> **Conclusion:** at *steady* 200 RPS, serverless costs ~2.5x — Lambda wins on spiky/low duty-cycle traffic, loses on sustained load. VMs vs Fargate ≈ wash on money; Fargate wins on patching/ops. Also note the fixed floor: the multi-AZ DB dominates all three — architecture cost talk that ignores the database is theater.

### Problem 6: The CTO challenge

You recommended the monolith (Problem 3). The CTO, ex-FAANG, responds: "Monoliths don't scale. Netflix runs microservices. I don't want to rebuild in two years." Write your reply: 4 sentences max, no strawmen, concede what's true.

**Expected output:**
```
A ≤4-sentence rebuttal handling scale, Netflix, and the rebuild fear.
```

> [!hint]- Hint
> Concede the 2-year concern, then show the exit path is pre-built.

> [!success]- Solution
> "You're right that we may outgrow it — that's why the modules have enforced boundaries and separate schemas, so extraction is a re-deployment, not a rewrite; the ADR names the exact triggers. Stateless monoliths scale horizontally a long way — Shopify and Stack Overflow served orders of magnitude more traffic than our 18-month projection on far simpler shapes. Netflix adopted microservices to coordinate two thousand engineers, and their platform investment was enormous; we have eight engineers and an audit to pass. I'd rather buy the distributed-systems tax when we have the revenue that justifies it — and with these seams, we can."

### Problem 7: Trade-off under constraint change

Mid-project, a new regulation requires all PII processing to run in-country, but your read-heavy product serves 6 countries from one region. Current: monolith + one RDS + CloudFront. Propose two compliant architectures with their trade-off table, and state which additional -ility just became top-3.

**Expected output:**
```
2 options + matrix + the newly promoted -ility.
```

> [!hint]- Hint
> Split by *data category*, not necessarily by whole deployments.

> [!success]- Solution
> Newly top-3: **compliance/data residency** (was implicit, now explicit and non-negotiable).
> **Option 1 — full replication:** deploy the whole stack per country (6× monolith+DB). ✅ simple mental model, full isolation; ❌ 6× infra+ops cost, cross-country features (global search) get hard, 6 deploy targets.
> **Option 2 — PII segregation:** keep global app + global non-PII DB; extract a PII service + small in-country DB per market (PII referenced by token). ✅ one main deployment, cost scales with PII volume only; ❌ new service boundary, every PII access becomes a network call, data modeling surgery.
> | (w: compliance ×3, cost ×2, delivery ×2, latency ×1) | Opt 1 | Opt 2 |
> |---|---|---|
> | Compliance | 5→15 | 4–5→13 (needs legal sign-off on tokenization) |
> | Cost | 2→4 | 4→8 |
> | Delivery speed | 3→6 | 3→6 |
> | Latency | 5→5 | 3→3 |
> | **Total** | **30** | **30** |
> Dead heat → decision hinges on legal's tokenization ruling and country count trajectory: more countries → Option 2 wins harder. This is the realistic outcome: matrices localize the *real* question; they don't always crown a winner.

### Problem 8: Full drill — timed solution architecture

45 minutes, from memory, produce a one-page solution architecture for: "A national pharmacy chain (300 stores) wants online ordering with in-store pickup: real-time stock per store, prescriptions require pharmacist approval, peak = flu season, existing stores run a legacy POS with a nightly export today." Deliverables: top-4 characteristics (measurable), context diagram, container-level design, 3 key trade-offs with your choices, and 2 risks with mitigations. Then compare against the solution.

**Expected output:**
```
One-page SA: characteristics, diagrams, 3 trade-offs, 2 risks.
```

> [!hint]- Hint
> The legacy nightly export is the hardest constraint — design around stale stock honestly.

> [!success]- Solution
> **Characteristics:** availability 99.9% during business hours; stock-freshness ≤ 15 min (NOT real-time — see trade-off 1); compliance (prescription data = health data, in-country, audited); elasticity 5x for flu season.
> **Context:** Customers (web/mobile) → Ordering System → integrates: legacy POS (per store), pharmacist approval app, payment provider, national e-prescription registry.
> **Containers:** SPA/mobile → API (modular monolith: catalog, orders, prescriptions, stock) → Postgres (multi-AZ); stock-sync service polling store POS gateways → stock cache (Redis) + events; pharmacist approval queue (work-queue UI); async workers (notifications). Cloud, containers on ECS; POS integration via per-store agent pushing deltas (stores have flaky links → queue + retry).
> **Trade-offs:** (1) *True real-time stock vs POS reality:* chose 15-min freshness + "reserve at order, confirm at pick" — honest availability beats fake precision; oversell handled by substitution/refund flow. (2) *Monolith vs microservices:* modular monolith — one retailer domain, moderate scale; prescriptions module isolated hardest (compliance boundary) as the first extraction candidate. (3) *Store connectivity:* store-side agent + async queue vs direct DB pulls — chose agent: survives offline stores, no VPN-to-300-stores nightmare.
> **Risks:** (1) POS vendor API limits → mitigation: pilot with 5 stores, fallback = accelerated nightly export + pessimistic stock buffer. (2) Flu-season spike overwhelms approval workflow (human bottleneck, not compute!) → mitigation: pharmacist workload dashboards, auto-routing between stores, pre-approval rules where legal.
> Grade yourself: did you catch that the pharmacist is the bottleneck and the POS is the constraint? Architecture is mostly about the ugly edges, not the boxes.
