---
title: "03 - Monolith Styles Drills"
date: 2026-07-11
tags:
  - software-architecture
  - layered
  - clean-architecture
  - hexagonal
  - practice
status: in-progress
parent: "[[Software Architecture Study]]"
---

# 03 — Monolith (Single-App) Styles Drills

> [!abstract] What you're drilling
> Layered, Clean, Hexagonal, Pipeline, Microkernel, Modular Monolith — recognizing, structuring, and choosing between them. See [[Software Architecture Study#7. Single-Application (Monolithic) Styles|Study §7]].

---

### Problem 1: Draw layered from memory

Draw the 4-layer architecture, mark closed layers, and trace a "GET /orders/42" request through it, naming a concrete artifact at each layer (Go or your language of choice).

**Expected output:**
```
ASCII diagram + 4 named artifacts (handler → service → repository → table).
```

> [!hint]- Hint
> Presentation → Business → Persistence → Database, each closed.

> [!success]- Solution
> ```
> [Presentation]  OrderHandler.GetOrder(w, r)      – parses ID, calls service
>       ↓ closed
> [Business]      OrderService.GetOrder(ctx, 42)   – authz check, business rules
>       ↓ closed
> [Persistence]   OrderRepository.FindByID(42)     – SQL, row→struct mapping
>       ↓
> [Database]      SELECT * FROM orders WHERE id=42
> ```
> Closed layers: the handler may never call the repository directly — that isolation is what lets you swap the persistence layer.

### Problem 2: Spot the sinkhole

In your layered app, `GET /countries` goes handler → service → repository → DB, and the service method is literally `return repo.FindAll()`. Is this the sinkhole anti-pattern? State the 80/20 rule and two remedies.

**Expected output:**
```
Diagnosis + rule of thumb + 2 remedies.
```

> [!hint]- Hint
> A few pass-throughs are normal; majority pass-throughs are not.

> [!success]- Solution
> One pass-through method isn't the anti-pattern — the **sinkhole** is when the *majority* (rule of thumb: if >80% of requests are pass-throughs, the layering isn't paying for itself). Remedies: (1) make some layers **open** for simple reads (document it!), (2) if most of the app is CRUD, drop to fewer layers or a simpler style — don't pay for isolation you don't use.

### Problem 3: Dependency Rule violation hunt

In a Clean Architecture codebase you find: (a) `entities/order.go` imports `"gorm.io/gorm"` for column tags, (b) `usecases/place_order.go` returns a `PlaceOrderHTTPResponse`, (c) `adapters/order_repo_pg.go` imports `usecases.OrderRepository` interface, (d) `usecases` imports `entities`. Which violate the Dependency Rule and how do you fix each?

**Expected output:**
```
Violations identified + fixes.
```

> [!hint]- Hint
> Dependencies point inward; inner circles know nothing of outer.

> [!success]- Solution
> - (a) **Violation** — the entity (innermost) knows the ORM (outermost). Fix: pure entity struct; a separate persistence model + mapper in the adapter layer.
> - (b) **Violation** — use case knows HTTP. Fix: use case returns a plain output DTO; the controller/presenter shapes HTTP.
> - (c) **Correct** — adapter *implements* an interface *defined by* the use-case layer: that's Dependency Inversion working as intended (source dependency points inward even though data flows outward).
> - (d) **Correct** — use cases may depend on entities (inward).

### Problem 4: Define the ports

Design hexagonal ports for a loan-approval service: inputs from REST API and a Kafka topic of applications; outputs to Postgres, a credit-bureau HTTP API, and an email provider. Name each port, classify driving/driven, and name its adapters.

**Expected output:**
```
Port list (interface names) + adapter list, classified.
```

> [!hint]- Hint
> Driving = who calls the core; driven = whom the core calls.

> [!success]- Solution
> **Driving ports** (core exposes): `LoanApplicationUseCase` { Submit, GetDecision } — adapters: `RestHandler`, `KafkaApplicationConsumer` (both call the same port — that's the payoff).
> **Driven ports** (core defines, infra implements): `ApplicationRepository` → `PostgresApplicationRepo`; `CreditScoreProvider` → `ExperianHTTPAdapter`; `DecisionNotifier` → `SESEmailAdapter`.
> Tests: a `FakeCreditScoreProvider` + in-memory repo let you test approval logic with zero infrastructure.

### Problem 5: Layered vs Clean — the honest call

Your team is building an internal admin CRUD tool (20 screens over a database, minimal logic). A colleague insists on full Clean Architecture "for quality." Make the counter-argument with 3 specific costs, then name one situation where you'd flip your own position.

**Expected output:**
```
3 costs + 1 flip condition.
```

> [!hint]- Hint
> What does each new CRUD screen cost in each style?

> [!success]- Solution
> Costs for CRUD-with-Clean: (1) every entity needs 3–4 representations (entity, persistence model, DTO, view model) + mappers — quadruple the code for zero behavioral gain; (2) new-screen velocity drops sharply — the tool's whole value is cheap screens; (3) the indirection obscures rather than protects: there's no domain logic to protect. Layered (or even 2-layer) wins.
> **Flip condition:** a real domain core emerges — e.g., the "admin tool" starts encoding approval workflows, pricing rules, or anything with business invariants that must be tested independently of the DB.

### Problem 6: Pipeline design

Design a pipeline architecture for processing uploaded CSV bank statements: parse → validate rows → normalize currencies → categorize transactions → write to DB + emit summary. Identify each filter's type (producer/transformer/tester/consumer) and state where you'd add a new "detect duplicates" step without touching others.

**Expected output:**
```
Filter chain with types + insertion point.
```

> [!hint]- Hint
> Filters are self-contained; pipes are one-way.

> [!success]- Solution
> ```
> [S3 upload event]  producer
>   → [CSVParser]        transformer  (bytes → rows)
>   → [RowValidator]     tester       (route invalid rows to reject sink)
>   → [Deduplicator]     tester       ◄ NEW: insert here, drop dupes
>   → [CurrencyNormalizer] transformer
>   → [Categorizer]      transformer
>   → [DBWriter]         consumer
>   → [SummaryEmitter]   consumer
> ```
> Because each filter only knows its input/output contract, inserting `Deduplicator` after validation touches no other filter — the core pipeline benefit.

### Problem 7: Microkernel fit

A tax-filing product must support 12 countries, each with different forms, rules, and deadlines, changing yearly. Argue microkernel: what's in the core, what's a plugin, what's the registry contract? What's the main risk?

**Expected output:**
```
Core responsibilities, plugin contents, contract sketch, 1 risk.
```

> [!hint]- Hint
> Core = stable happy path; plugins = volatile variation.

> [!success]- Solution
> **Core:** user accounts, document storage, filing workflow engine, payment, plugin registry.
> **Plugins:** one per country — form definitions, validation rules, tax calculation, submission adapter to that country's tax authority.
> **Contract:** `TaxPlugin { CountryCode(); Forms(year); Validate(return); Calculate(return); Submit(return) }` + manifest (versions, supported years).
> **Risk:** the core contract must anticipate variation — if Brazil needs a workflow step the contract doesn't allow, you're changing the core anyway. Mitigate by designing the contract from the 2–3 *most different* countries first.

### Problem 8: Modular monolith boundaries

Design the module structure for a job-board (employers post jobs, candidates apply, search, subscription billing) as a modular monolith. Show module boundaries, what each module's public API looks like, and 2 rules you'd enforce with fitness functions.

**Expected output:**
```
Module tree + API surface + 2 enforced rules.
```

> [!hint]- Hint
> Module = own package, own tables, public facade only.

> [!success]- Solution
> ```
> jobboard/
>   jobs/        (posting, moderation)       api: PostJob, GetJob, SearchJobs
>   applications/(apply, status)             api: Apply, ListForJob, ListForCandidate
>   candidates/  (profiles)                  api: GetProfile, UpdateProfile
>   billing/     (plans, invoices)           api: CanPost(employerID), Subscribe
>   identity/    (auth, roles)               api: CurrentUser, Authorize
> ```
> Each module: `internal/` guts + one public facade package; **own tables** (`jobs_*`, `applications_*`), cross-module access only via facades (never each other's tables).
> **Fitness functions:** (1) import rule — `jobs/internal/**` unimportable outside `jobs/` (Go enforces via `internal/`; elsewhere use arch tests); (2) DB rule — CI test greps migrations/queries: module X's code may not reference module Y's table prefix.
> Payoff: `billing` can be extracted to a service later by swapping the facade for an HTTP client.

### Problem 9: Style recognition speed round

Name the style from the smell: (a) "to add a feature we write a plugin JAR and drop it in /ext", (b) "every request passes through 4 layers even for static lookups", (c) "the DB is a detail — look, we swapped Postgres for Mongo in tests via an interface", (d) "gunzip | parse | filter | aggregate | report", (e) "one deployable, but the checkout team never touches inventory's packages."

**Expected output:**
```
5 style names.
```

> [!hint]- Hint
> One is an anti-pattern symptom, not a style.

> [!success]- Solution
> (a) Microkernel • (b) Layered — with sinkhole symptoms • (c) Clean/Hexagonal (Dependency Rule + ports) • (d) Pipeline • (e) Modular monolith (domain-partitioned).
