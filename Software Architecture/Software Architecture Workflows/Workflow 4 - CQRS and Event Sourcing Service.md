---
title: "Workflow 4 - CQRS and Event Sourcing Service"
date: 2026-07-11
tags:
  - software-architecture
  - cqrs
  - event-sourcing
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 4 — CQRS + Event Sourcing: A Wallet Service

> [!abstract] Project
> Build **"Kapao"**, the wallet/ledger service inside the Sarn marketplace (buyer credits, seller payouts, refunds, dispute holds) — the textbook-correct home for CQRS + event sourcing ([[05 - Communication and Data Drills#Problem 8 Event-source or not?|why this domain]]). Runs locally with Go/your language + Postgres; no framework required — building it raw is the pedagogy. Deployable later as one service inside [[Workflow 3 - Microservices and Event-Driven on Cloud|W3]]'s landscape.

**Architecture:**

```
  commands                            queries
     │                                   ▲
┌────▼──────────────┐             ┌──────┴──────────┐
│  Command handler  │             │   Query API     │
│  load events →    │             │ (reads views    │
│  rebuild aggregate│             │  only, no logic)│
│  → validate → new │             └──────▲──────────┘
│  events → append  │                    │
└────┬──────────────┘             ┌──────┴──────────┐
     │ append (OCC)               │   Read views    │
┌────▼──────────────┐  projector  │ wallet_balances │
│   EVENT STORE     │────────────►│ tx_history      │
│ (append-only,     │  (poll/     │ daily_stats     │
│  source of truth) │   notify)   └─────────────────┘
└────┬──────────────┘
     │ publisher
     ▼ Kafka: wallet.events (for the rest of the system)
```

---

## Step 1 — Model the events

**Do:** define the wallet aggregate's events and commands. Distinguish them sharply ([[05 - Communication and Data Drills#Problem 2 Event or command?|drill]]). Include disputes (hold/release) and the invariant the aggregate protects.

> [!success]- Solution
> **Commands (can fail):** `OpenWallet`, `Deposit`, `Withdraw`, `HoldFunds`, `ReleaseHold`, `SettleHold`.
> **Events (facts, past tense, cannot fail):** `WalletOpened{owner}`, `FundsDeposited{amount, ref}`, `FundsWithdrawn{amount, ref}`, `WithdrawalRejected{reason}` (yes — rejections can be events too: auditors love them), `FundsHeld{disputeID, amount}`, `HoldReleased`, `HoldSettled`.
> **Invariant:** `available = deposits − withdrawals − active_holds ≥ 0` — enforced in exactly one place: the command handler, against state rebuilt from events. Events carry money as integer minor units (satang), never floats; every event has `{walletID, seq, occurredAt, actor, causationID}`.

## Step 2 — Build the event store on Postgres

**Do:** design the table + append operation with optimistic concurrency control (two concurrent withdrawals must not both succeed on a balance that only covers one).

> [!success]- Solution
> ```sql
> CREATE TABLE events (
>   wallet_id   UUID        NOT NULL,
>   seq         BIGINT      NOT NULL,        -- per-aggregate sequence
>   type        TEXT        NOT NULL,
>   payload     JSONB       NOT NULL,
>   occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
>   PRIMARY KEY (wallet_id, seq)             -- ◄ the whole trick
> );
> ```
> **Append = OCC:** handler reads events (say up to seq 41), rebuilds state, validates, appends new event with `seq = 42`. A concurrent writer also targeting 42 hits the primary-key violation → **retry**: reload (now sees 42), re-validate against fresh balance, append at 43 — or reject if funds no longer suffice. The PK turns a race condition into a clean conflict. No locks held during validation; conflict cost paid only on actual contention.

## Step 3 — Command handler + aggregate rebuild

**Do:** implement `Withdraw` end-to-end: load, fold, validate, append. Then add snapshots for wallets with 10k+ events.

> [!success]- Solution
> ```go
> func (h *Handler) Withdraw(ctx context.Context, cmd Withdraw) error {
>     events, err := h.store.Load(ctx, cmd.WalletID)         // all events (or snapshot+tail)
>     if err != nil { return err }
>     w := ReplayWallet(events)                              // pure fold: state = f(events)
>     if w.Available() < cmd.Amount {
>         return h.store.Append(ctx, cmd.WalletID, w.Seq+1,  // even rejection = event
>             WithdrawalRejected{cmd.Amount, "insufficient"})
>     }
>     return h.store.Append(ctx, cmd.WalletID, w.Seq+1,      // OCC-protected
>         FundsWithdrawn{cmd.Amount, cmd.Ref})               // retry-on-conflict wrapper
> }
> ```
> `ReplayWallet` is a pure function — trivially unit-tested with event fixtures, no DB. **Snapshots:** every N events store `(wallet_id, seq, state_json)`; load = latest snapshot + events after it. Snapshots are a cache — deletable, rebuildable, never the source of truth.

## Step 4 — Projections (the CQRS read side)

**Do:** build the projector that maintains `wallet_balances` and `tx_history` views, tracking its own position, restartable, and able to rebuild from scratch.

> [!success]- Solution
> Projector loop: read events after `checkpoint` (global ordering via a `BIGSERIAL global_seq` column added to the events table) → apply to view tables → update checkpoint **in the same transaction** (atomic apply+checkpoint = exactly-once *effect* on the view). Poll or LISTEN/NOTIFY.
> ```sql
> CREATE TABLE wallet_balances (wallet_id UUID PRIMARY KEY,
>   available BIGINT, held BIGINT, updated_seq BIGINT);
> CREATE TABLE projector_checkpoints (name TEXT PRIMARY KEY, position BIGINT);
> ```
> **Rebuild** = the superpower: `TRUNCATE` views, checkpoint→0, replay all history. New feature "monthly seller statements"? A new projector replays two years of events and the feature ships with full history — impossible in state-based CRUD. Queries (`GetBalance`, `GetHistory`) read views only: single-row reads, no folding, fast.

## Step 5 — Consistency, UX, and the API seam

**Do:** the projector lags ~200ms. Design the API so clients aren't confused: what does `Withdraw`'s response contain, and how do you offer read-your-own-writes ([[05 - Communication and Data Drills#Problem 7 The stale-read UX problem|drill]])?

> [!success]- Solution
> `POST /withdraw` → `200 {walletID, seq: 43, newAvailable: 1500}` — the command handler *just computed* the authoritative state; return it. The client updates its UI from the response (optimistic-but-actually-authoritative). For subsequent GETs: client passes `?min_seq=43`; if the view's `updated_seq < 43`, the query API either waits briefly, falls back to folding that one wallet's tail events, or returns `202 retry-after` — pick one policy and document it. External consumers get `wallet.events` via outbox-style publisher from the event table (the event store *is* an outbox — one dual-write problem you get solved for free).

## Step 6 — Schema evolution & GDPR/PDPA reality

**Do:** two years in: (a) `FundsDeposited` needs a `channel` field; (b) `FundsDeposited.ref` format must change meaning; (c) a user invokes right-to-erasure but events are immutable. Solve all three.

> [!success]- Solution
> - (a) **Additive** = easy: optional field, defaulted on replay of old events. This must be 95% of your evolution.
> - (b) **Never mutate meaning in place.** New event type/version `FundsDepositedV2`; replay handles both forever — or run a one-time *upcast* (transform-on-read) layer. Old events are history; history doesn't get edited.
> - (c) **Crypto-shredding:** PII in events is encrypted with a per-user key stored outside the event store; erasure = destroy the key → events remain (ledger stays consistent, sums still balance) but PII is unrecoverable. Design this *before* launch — retrofitting encryption onto an immutable log is misery. Amounts/IDs stay plaintext (not PII, needed for audit).

## Step 7 — The honest scorecard

**Do:** write the trade-off retrospective: what Kapao gained from ES+CQRS, what it pays forever, and name three neighboring services that must NOT copy this pattern.

> [!success]- Solution
> **Gained:** audit trail regulators accept as-is; disputes resolved by replaying facts; new read models from history; OCC gives correct concurrent-money behavior without lock gymnastics; testing is pure-function pleasant.
> **Pays forever:** every schema change is an upcasting decision; onboarding devs takes weeks ("where's the UPDATE statement?"); ops must guard the events table (it IS the company's money); projector lag is a standing support-education topic.
> **Must not copy it:** Catalog (products are current-state — nobody audits description edits), Notifications (fire-and-forget, zero audit value), user Profiles ([[05 - Communication and Data Drills#Problem 8 Event-source or not?|drill 8's verdict]]). Event sourcing is a scalpel for ledger-shaped domains, not a house style. Write the ADR saying exactly this, so future teams inherit the *reasoning*, not just the pattern.

---

## Extensions

1. Add `TransferBetweenWallets` — two aggregates, one business action: saga across aggregates with hold→deposit→settle ([[05 - Communication and Data Drills#Problem 4 Design the saga|saga drill]] applied inward).
2. Load-test: 1k concurrent withdrawals on one wallet vs 1k wallets — measure OCC conflict rates; observe why per-aggregate streams scale and hot aggregates don't.
3. Build a second projector `daily_stats` and rebuild it from genesis while the system serves traffic.
4. Swap the Postgres event store for EventStoreDB or Kafka-as-log; write one page on what each change breaks.
