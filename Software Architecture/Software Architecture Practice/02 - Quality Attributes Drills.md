---
title: "02 - Quality Attributes Drills"
date: 2026-07-11
tags:
  - software-architecture
  - quality-attributes
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 02 — Quality Attributes ("-ilities") Drills

> [!abstract] What you're drilling
> Extracting characteristics from requirements, making them measurable, spotting conflicts, choosing the top 3. See [[Software Architecture Study#5. Quality Attributes — the "-ilities"|Study §5]].

---

### Problem 1: Name that -ility

Match each statement to exactly one characteristic: (a) "System must survive a whole datacenter outage", (b) "New pricing rules ship without redeploying", (c) "Search results in under 200ms", (d) "Handle 50x traffic during World Cup finals, then shrink", (e) "A junior dev can add a report type in a day", (f) "Auditors can reconstruct any account balance at any past date."

**Expected output:**
```
6 correct attribute names.
```

> [!hint]- Hint
> One is auditability/recoverability-adjacent, one is elasticity not scalability.

> [!success]- Solution
> (a) fault tolerance / availability (multi-AZ) • (b) extensibility (or configurability) • (c) performance (latency) • (d) **elasticity** (spike + shrink — not mere scalability) • (e) maintainability/extensibility • (f) auditability.

### Problem 2: Make them measurable

Rewrite these useless requirements as measurable ones: (a) "The system should be fast", (b) "The system should be reliable", (c) "The system should scale", (d) "The system should be secure."

**Expected output:**
```
4 statements with numbers, percentiles, and conditions.
```

> [!hint]- Hint
> Always: metric + percentile + load condition + target.

> [!success]- Solution
> - (a) p99 API latency < 500ms and p50 < 100ms at 1,000 RPS sustained.
> - (b) 99.9% monthly availability (≤ 43.8 min downtime); no single-instance failure causes user-visible errors > 30s.
> - (c) Linear cost scaling to 10x current load with no code change; onboard 10k new users/day without degradation.
> - (d) All data encrypted in transit (TLS 1.2+) and at rest; authZ enforced per-endpoint (verified by automated tests); OWASP Top 10 scan gate in CI; secrets never in code (CI check).

### Problem 3: Availability math

Compute allowed downtime per month for 99%, 99.9%, 99.99%. Then: your app depends *synchronously* on 3 services, each 99.9%. What's your best-case availability, and what's the architectural lesson?

**Expected output:**
```
Downtime figures + composed availability + one-sentence lesson.
```

> [!hint]- Hint
> Serial dependencies multiply.

> [!success]- Solution
> Per month (~730h): 99% = 7.3h; 99.9% = 43.8 min; 99.99% = 4.4 min.
> Composed: 0.999³ ≈ **99.7%** (~2.2h/month) — you can never exceed your weakest synchronous chain. **Lesson:** every added sync dependency lowers your ceiling; use async, caching, or graceful degradation to break the chain.

### Problem 4: Spot the conflict

For each pair, explain the tension in one sentence and name one mitigation: (a) security vs performance, (b) scalability vs consistency, (c) deployability vs reliability, (d) cost vs availability.

**Expected output:**
```
4 tensions + 4 mitigations.
```

> [!hint]- Hint
> CAP theorem lives in (b).

> [!success]- Solution
> - (a) Encryption/authz checks add latency → mitigate: TLS termination at LB, cache authz decisions (short-TTL), hardware acceleration.
> - (b) Replicating for scale forces choosing availability or consistency under partition (CAP) → mitigate: eventual consistency where business tolerates it, strong only where it doesn't (money).
> - (c) More releases = more change-induced incidents → mitigate: small batches, canary deploys, automated rollback (deployability done *right* increases reliability).
> - (d) Each extra 9 multiplies infra (multi-AZ→multi-region) and eng cost → mitigate: buy 9s only where the business case exists; degrade gracefully instead.

### Problem 5: Pick the top 3

An online exam platform for universities: 10k students take an exam simultaneously at 9:00, results private, cheating detection, universities pay per exam. Choose the top-3 characteristics, and explicitly name 2 you're *sacrificing*.

**Expected output:**
```
Top 3 with justification + 2 explicit sacrifices.
```

> [!hint]- Hint
> What happens at 8:59 → 9:01? What's the worst headline?

> [!success]- Solution
> **Top 3:** (1) **Elasticity** — load goes 0 → 10k in one minute at 9:00 (pre-warm or serverless); (2) **Availability during exam windows** — a crash mid-exam is existential (note: *windowed* availability, cheaper than 24/7); (3) **Security/privacy** — exam integrity + student data (headline risk).
> **Sacrificed:** time-to-market for new features (exam season fixed, quality first), cost-efficiency off-season (pre-warmed capacity), and general 24/7 availability (maintenance windows fine at 3am).

### Problem 6: Implicit characteristics

A requirement doc for a payment-processing feature says nothing about security, compliance, or auditability. As architect, list 4 characteristics you must include anyway, and their source (law/standard/common sense).

**Expected output:**
```
4 implicit characteristics + sources.
```

> [!hint]- Hint
> Some characteristics are never in the requirements because "everyone knows" — until they're missing.

> [!success]- Solution
> 1. **Security** — PCI-DSS if touching card data (never store PAN/CVV; tokenize via provider).
> 2. **Auditability** — financial regulations + dispute resolution: immutable transaction log.
> 3. **Data integrity/consistency** — money must balance: idempotent operations, exactly-once *effects*, reconciliation jobs.
> 4. **Privacy/compliance** — GDPR/PDPA for cardholder personal data, data residency.
> Implicit characteristics are the architect's responsibility precisely because nobody writes them down.

### Problem 7: Fitness functions

Write 3 concrete, automatable fitness functions: (a) enforce that the domain layer never imports the persistence layer, (b) p99 latency budget in CI, (c) no service calls another service's database.

**Expected output:**
```
3 checks with the tool/mechanism you'd use.
```

> [!hint]- Hint
> Think dependency-analysis tests, load tests as pipeline gates, and network policies / lint rules.

> [!success]- Solution
> - (a) Architecture test: ArchUnit (Java) / `depguard`+`go-arch-lint` (Go) / `dependency-cruiser` (TS) rule: `domain/** must not import persistence/**` — runs as a unit test, fails the build.
> - (b) CI stage runs k6/Gatling against a staging deploy: `http_req_duration p(99) < 500ms` threshold fails the pipeline.
> - (c) DB credentials per service in secrets manager (no shared creds) + network policy/security group allowing DB port only from its owning service; optional nightly query of `pg_stat_activity` client IPs, alert on strangers.

### Problem 8: Characteristics drive structure

Same functional requirements (a URL shortener), two different characteristic sets: (A) internal tool, 50 users, availability 99%, 1 dev; (B) public, 100k redirects/sec, p99 < 20ms globally, 99.99%. Sketch the architecture for each and list what changed *purely because of -ilities*.

**Expected output:**
```
Two architectures + a diff list.
```

> [!hint]- Hint
> Functionality identical; everything else differs.

> [!success]- Solution
> **(A):** one small VM: nginx + app + SQLite/Postgres. Backups nightly. Done.
> **(B):** anycast CDN/edge (redirects served from edge KV, e.g., CloudFront Functions/Workers KV), write path → API → replicated store (DynamoDB global tables / Redis + persistent store), cache-aside, multi-region, observability stack, rate limiting.
> **Diff caused by -ilities alone:** edge caching layer, geo-replication, storage choice (KV vs relational), multi-region failover, dedicated observability, rate-limiting — zero new *features*. This is the whole point: architecture serves characteristics, not features.

### Problem 9: The negotiation

Product wants: 99.99% availability, sub-100ms p99 globally, full feature parity in 3 months, on a $500/month infra budget with 2 engineers. Write the 4-sentence pushback you'd give, offering two coherent packages.

**Expected output:**
```
A short trade-off negotiation with 2 packages.
```

> [!hint]- Hint
> Don't say no — price each -ility.

> [!success]- Solution
> "These four constraints can't coexist: 99.99% requires multi-region redundancy that alone exceeds $500/mo, and global sub-100ms needs edge infrastructure and time we don't have with 2 engineers in 3 months. **Package A (launch-focused):** 99.9%, sub-100ms in SEA only, 3 months, ~$500/mo — we buy headroom to go global later. **Package B (quality-focused):** 99.95%+ and global latency targets, but 6 months and ~$3k/mo. Which -ilities does the business actually need first?"
