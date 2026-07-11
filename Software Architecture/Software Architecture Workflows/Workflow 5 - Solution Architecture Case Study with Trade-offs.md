---
title: "Workflow 5 - Solution Architecture Case Study with Trade-offs"
date: 2026-07-11
tags:
  - software-architecture
  - solution-architecture
  - trade-offs
  - cloud
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 5 — Solution Architecture Case Study: Comprehensive Trade-off Analysis

> [!abstract] Project
> The capstone. You are the solution architect for **"MediLink"** — a health-tech company replacing its clinic-management vendor product with its own platform: appointment booking, telemedicine video, e-prescriptions, insurance claims, for 400 clinics nationwide. Regulated health data, real money, real deadlines. You'll run a full SA engagement: discovery → candidates → deep trade-off analysis (compute, data, communication, cloud) → final architecture → executive defense. This workflow *is* the interview-grade deliverable; treat every step as producing a document.

**Final architecture (built up through the steps):**

```
   Clinic staff ─┐                         Patients ─┐
   (web portal)  │                         (mobile)  │
                 ▼                                   ▼
        ┌─────────────────────────────────────────────────┐
        │            API Gateway + WAF  (authn, audit)    │
        └──────┬──────────────┬──────────────┬────────────┘
        ┌──────▼─────┐ ┌──────▼──────┐ ┌─────▼──────────┐
        │ Core API   │ │ Claims svc  │ │ Telemed signal │
        │ (modular   │ │ (isolated:  │ │ svc (websocket)│
        │ monolith:  │ │ insurer     │ └─────┬──────────┘
        │ scheduling,│ │ integrations│       │ media via managed
        │ patients,  │ │ + batch)    │       ▼ WebRTC (Chime/Twilio)
        │ rx)        │ └──────┬──────┘
        └──┬─────┬───┘        │
   ┌───────▼─┐ ┌─▼─────┐ ┌───▼────┐    Events: SQS/SNS
   │Postgres │ │ Redis │ │Claims  │    (outbox from Core)
   │multi-AZ │ │       │ │DB      │    Workers: notifications,
   │(PHI:    │ └───────┘ └────────┘    reminders, exports
   │encrypted│
   └─────────┘
   Audit log store (WORM)  ·  In-country region  ·  IaC everything
```

---

## Step 1 — Discovery: requirements, constraints, characteristics

**Do:** from the brief, produce the characteristics workbook. Brief: 400 clinics, ~8k staff users, ~2M patients; clinics work 08:00–20:00; telemedicine is the growth bet; insurance claims batch to 5 insurers (SOAP/sFTP, sadly real); health-data law: in-country storage, access audit, 10-year retention; company: 25 engineers, 12-month deadline to leave the vendor, board is cost-sensitive.

> [!success]- Solution
> | Characteristic | Target | Driver |
> |---|---|---|
> | Compliance | in-country region; PHI encrypted at rest+transit; every PHI access audit-logged (immutable, 10y) | law |
> | Availability | 99.9% 08:00–20:00 clinic days; telemed 99.95% during sessions | clinic ops; a dropped consult = clinical risk |
> | Security | role-based (doctor/nurse/admin/patient), least-privilege, pen-tested | PHI |
> | Migration safety | vendor exit with zero patient-record loss; parallel-run period | the actual project risk |
> | Time-to-market | 12 months, phased | vendor contract end |
> | Cost | ≤ $15k/mo at full rollout | board |
> | Performance | booking p95 < 500ms; video join < 3s | UX |
>
> Notice **migration safety** — never in the -ilities catalog but the #1 real risk of a replacement project. Discovery means finding the characteristics the brief didn't name.

## Step 2 — Compute trade-off: monolith vs services vs serverless

**Do:** full weighted matrix across three compute candidates for the core platform, sensitivity analysis included ([[06 - Trade-off Analysis Drills|drills 1–2]] method).

> [!success]- Solution
> Candidates: (A) modular monolith + 2 satellite services; (B) 8-service microservices; (C) serverless-first (Lambda + managed everything).
> | Criterion (w) | A | B | C |
> |---|---|---|---|
> | 12-mo delivery, 25 devs (×3) | 5→15 | 3→9 | 4→12 |
> | Compliance/auditability (×3) | 5→15 | 3→9 | 3→9 |
> | Availability targets (×2) | 4→8 | 4→8 | 4→8 |
> | Cost @ scale (×2) | 4→8 | 3→6 | 3→6 (steady load hurts FaaS — [[06 - Trade-off Analysis Drills#Problem 5 Cost model duel|cost duel]]) |
> | Team fit/ops maturity (×2) | 5→10 | 2→4 | 3→6 |
> | **Total** | **56** | **36** | **41** |
>
> **A wins decisively.** But the matrix's real yield is *which parts escape the monolith*: Claims (insurer SOAP/sFTP integrations = volatile, batch, failure-prone → isolate the blast radius) and Telemed signaling (websockets, different availability class, spiky). **Sensitivity:** if telemed 10x's (the growth bet), only the telemed service scales — the decision is *robust to the business's own optimism*, which is exactly what you want to show the board.

## Step 3 — Data trade-offs: storage, PHI, audit

**Do:** decide: one DB or several? relational or document for patient records? where does the immutable audit trail live? Each with the rejected alternative and why.

> [!success]- Solution
> - **Postgres (managed, multi-AZ, in-country) as system of record.** Rejected MongoDB-for-records: clinical data is deeply relational (patient↔encounters↔prescriptions↔claims) and the auditors' tooling speaks SQL. JSONB columns give document flexibility where forms vary per clinic.
> - **Claims service gets its own DB** (schema autonomy for insurer-format staging; failure isolation from clinical ops). Core + telemed share the main DB (telemed stores only session metadata — media never touches disk: retention liability).
> - **Audit trail: append-only store with object-lock/WORM** (e.g., partitioned audit table + S3 Object Lock exports; or QLDB-style ledger). *Every* PHI read/write emits an audit event via the outbox — synchronous audit writes would double p95 latency (measured trade-off: async with ≤ 5s durability lag accepted, documented in the ADR, cleared with legal).
> - **Encryption:** at-rest via KMS + column-level for the highest-sensitivity fields; crypto-shredding strategy for right-to-erasure vs 10-year retention conflicts (retention wins for clinical records under the law — legal sign-off recorded).

## Step 4 — Communication & integration trade-offs

**Do:** decide sync vs async per flow: booking, reminders, claims submission, telemed signaling, insurer callbacks. Then design the vendor-migration data flow.

> [!success]- Solution
> | Flow | Choice | Why |
> |---|---|---|
> | Booking/patient lookup | sync REST | user-blocking, needs answer ([[05 - Communication and Data Drills#Problem 1 Sync or async?|drill 1]] pattern) |
> | Reminders/notifications | async (outbox→SQS→workers) | side-effects; retryable |
> | Claims submission | async batch + orchestrated saga per claim | insurers are slow/flaky; compensations = resubmit/flag-for-human |
> | Telemed signaling | websocket svc; media via managed WebRTC (Chime/Twilio) | never build media servers with 25 devs — buy the hardest real-time problem |
> | Insurer callbacks | webhook endpoint → queue → claims svc | absorb their retries/duplicates; idempotency keys |
>
> **Migration:** strangler-fig against the vendor: nightly exports → staging → reconciliation reports (count/checksum per entity) → clinics migrate in waves of 20 with a 2-week parallel-run (writes to both, reads from new, daily diff reports) → vendor read-only → cutoff. The reconciliation tooling is a *deliverable*, not a script — it's what makes "zero record loss" a claim you can prove.

## Step 5 — Cloud architecture & cost model

**Do:** produce the deployment architecture (region, network, services) and a cost estimate against the $15k/mo ceiling.

> [!success]- Solution
> Single in-country region (law kills multi-region debates — convenient!), 3 AZs. VPC: public (ALB/WAF) / private-app (ECS Fargate: core, claims, telemed, workers) / private-data (RDS multi-AZ, ElastiCache, no internet path). API Gateway/WAF at edge; all PHI endpoints behind clinic-staff SSO (OIDC) or patient auth. IaC (Terraform) from day one — for a regulated system, *reviewable infrastructure changes are a compliance feature*.
> **Ballpark:** Fargate fleet ~$2.5k · RDS r6g.xlarge multi-AZ + replica ~$2k · Redis ~$300 · ALB/WAF/API GW ~$500 · SQS/SNS ~$100 · managed WebRTC ~$3–4k (usage! the scaling cost lever) · observability ~$800 · S3/backup/audit ~$500 · NAT/misc ~$400 ≈ **$10–11k/mo** — headroom vs the $15k ceiling, with telemed media as the elastic cost to watch (per-minute pricing scales with the growth bet — flag to the board as COGS, not infra overhead).

## Step 6 — Risk register & the parts that can go wrong

**Do:** name the top 5 risks with likelihood/impact and mitigations. At least two must be non-technical.

> [!success]- Solution
> | Risk | L×I | Mitigation |
> |---|---|---|
> | Insurer integrations slip (5 bespoke SOAP/sFTP) | H×H | start month 1 (not month 9); claims svc isolation; manual-submission fallback path keeps clinics operational |
> | Vendor obstructs data export | M×H | contract review now; escrow clause; scraping-grade fallback exporter; keep parallel-run window generous |
> | Clinic staff reject new UX | M×H | 5 pilot clinics co-design; train-the-trainer; feature-parity checklist signed by clinic ops, not engineering |
> | Telemed cost blowout on success | M×M | per-minute cost dashboards; renegotiate committed-use at 6 months; codec/quality tiers |
> | Audit-log gap discovered late | L×H | audit middleware built into the gateway+ORM layer in month 1; compliance review at each phase gate, not the end |

## Step 7 — The executive defense

**Do:** write the one-page decision summary for the board: recommendation, the two roads not taken and why, what would change your mind, and the phased plan. This page *is* the solution architect's product.

> [!success]- Solution
> **Recommendation:** modular-monolith core with two isolated services (claims, telemedicine) on managed cloud in-country; buy video infrastructure; strangler-fig migration in clinic waves; $10–11k/mo run cost at full rollout.
> **Roads not taken:** *Microservices* — organizational overhead our 25 engineers would pay before rollout even completes; our seams (claims, telemed) are already isolated where the risk lives; the modular core keeps extraction cheap if we triple headcount. *Serverless-first* — steady clinic-hours load fits containers' cost curve better, and the compliance story (long-lived audit middleware, VPC-pinned data paths) is simpler to evidence.
> **What would change my mind:** telemed sessions > 40% of revenue (extract + regionalize media), engineering > 60 (service-based split along the module seams), a second country (data-residency architecture per [[06 - Trade-off Analysis Drills#Problem 7 Trade-off under constraint change|drill 7]]).
> **Phases:** M1–3 walking skeleton + audit spine + insurer #1 integration; M4–6 pilot 5 clinics parallel-run; M7–9 waves ×20 clinics + telemed GA; M10–12 full rollout + vendor exit. Each phase gates on reconciliation + compliance review.
> *Every claim above traces to a matrix, a cost line, or an ADR — that traceability, not the diagram, is what makes it solution architecture.*

---

## Extensions

1. The board asks: "Could we run this on-prem in the clinics instead of cloud?" Produce the trade-off table (hint: 400 sites × ops = the answer, but *show* it).
2. Redo Step 2's matrix assuming 100 engineers and no deadline — defend whatever wins.
3. Write ADR-00X for "buy vs build telemedicine media" including the exit strategy if the vendor 4x's prices.
4. Take a real system you know and run Steps 1→7 on it in 90 minutes. This template is reusable for actual SA interviews and RFP responses.
