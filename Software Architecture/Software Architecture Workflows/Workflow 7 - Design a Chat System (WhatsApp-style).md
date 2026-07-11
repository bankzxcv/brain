---
title: "Workflow 7 - Design a Chat System (WhatsApp-style)"
date: 2026-07-11
tags:
  - software-architecture
  - system-design
  - chat
  - websocket
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 7 — Famous Case: Chat System (WhatsApp-style)

> [!abstract] Project
> Design 1:1 and group messaging with delivery/read receipts, online presence, and offline delivery. Where [[Workflow 6 - Design a News Feed (Twitter-style)|the feed]] was a read-scaling problem, chat is a **stateful-connection and delivery-guarantee** problem: millions of open sockets, and "the message must arrive exactly once, in order" — on phones that die, roam, and go offline mid-send.

**Architecture (target):**

```
 mobile A ══websocket══► Chat server 17 ─┐
                         (holds A's conn)│   session registry
 mobile B ══websocket══► Chat server 42 ─┤   (Redis): user → server
                         (holds B's conn)│
                                         ▼
                              ┌─────────────────┐
                              │  Message service │──► Message store
                              │  (route/persist) │    (Cassandra-style,
                              └───────┬─────────┘     partition: convo_id)
                                      │
              offline? ──► push notification svc (FCM/APNs)
              group msg ──► fan-out to member sessions
              presence ──► presence svc (heartbeats, pub/sub)
              media    ──► blob storage + CDN (URL in message)
```

---

## Step 1 — Requirements & the defining constraints

**Do:** scope it (1:1, groups ≤ 500, receipts, presence, media pointers; skip calls/E2E crypto for now). 500M DAU, 40 messages/user/day. Compute message QPS, then name the two constraints that make chat architecturally *unlike* a web app.

> [!success]- Solution
> 500M × 40 / 86,400 ≈ **230k msg/s** (peak ~700k/s). Tiny payloads (~1KB) — bandwidth is trivial; *connection count* is not.
> The two defining constraints: (1) **servers hold state** — a websocket per online user (100M+ concurrent connections); the stateless-app-tier playbook from [[Workflow 2 - Modular Monolith on a Cloud VM#Step 3 — Make the app tier stateless|W2]] doesn't apply to the connection layer; (2) **delivery semantics are the product** — ordering within a conversation, exactly-once *appearance*, offline queueing. Characteristics ranking: reliability of delivery > latency (< 200ms feels instant) > availability > everything else. Consistency here is *per-conversation ordering*, not global — a crucial narrowing.

## Step 2 — Why not HTTP? Choosing the connection model

**Do:** compare polling, long-polling, websocket for message delivery at this scale ([[Software Architecture Study#8.0 Synchronous & Asynchronous Communication|sync/async]] applied to the client edge). Decide, and size the connection fleet.

> [!success]- Solution
> Polling at 5s: 500M × 0.2 req/s = 100M req/s of mostly-empty responses — absurd. Long-polling halves the waste, still reconnect-storms. **Websocket wins**: one persistent duplex connection, server pushes instantly, heartbeats double as presence. Cost: stateful servers, LB affinity, harder deploys (draining = disconnecting users).
> Fleet: a tuned server holds ~500k–1M idle connections (memory-bound: ~10KB/conn) → ~100–200 chat servers for 100M concurrent. The **session registry** (Redis: `user_id → server_id`, TTL-heartbeat) is how anyone finds anyone — it becomes tier-0 infrastructure: shard it, replicate it, and design for it being briefly wrong (server crashed, registry stale → route fails → fallback to offline path; self-heals on reconnect).

## Step 3 — The message flow: A sends to B

**Do:** trace the happy path A→B on different servers, then B-offline. Mark every point where a message could be lost, and the mechanism that prevents it.

> [!success]- Solution
> ```
> A ──ws──► Server17: assign msg_id (per-convo monotonic seq)
>            persist to store (ack only AFTER durable)   ◄── loss point 1: ack-before-persist
>            ──► registry lookup: B @ Server42
>            ──► Server42 ──ws──► B
>            B client acks ──► mark delivered ──► receipt to A
> offline:   registry miss → enqueue in B's inbox (already persisted)
>            → push notification via FCM/APNs
>            B reconnects → syncs inbox since last_seen_seq
> ```
> Loss points & cures: (1) server crash before persist → client retries (msg kept locally with client-generated UUID; server dedupes = **idempotency**, [[04 - Distributed Styles Drills#Problem 7 Idempotent consumer|the drill]] again); (2) delivery crash after persist → B's *pull-on-reconnect* (sync since last seq) is the true guarantee — push is just an optimization of pull. That inversion — **persistent inbox as truth, live push as accelerator** — is the core chat insight: the system is a per-user message log with a fast path, i.e., a cousin of the [[Workflow 4 - CQRS and Event Sourcing Service|event store]].

## Step 4 — Ordering and the message ID

**Do:** why can't you use server timestamps for ordering? Design the ID/sequence scheme for 1:1 and group conversations, and state what "ordering" you deliberately don't guarantee.

> [!success]- Solution
> Server clocks skew (NTP ≈ ms–s drift across 200 servers) — two messages 5ms apart can invert. **Per-conversation sequence**: the partition owner (message service shard for that convo_id) assigns monotonic seq numbers; clients render by seq, detect gaps (seq 7 after 5 → fetch 6 → *ordered exactly-once appearance* even over an unreliable pipe).
> Not guaranteed: **cross-conversation ordering** (message in chat X vs chat Y) — nobody can perceive it, so nobody pays for it; and **global timestamps** are display-only. This is the narrowing from Step 1 paying off: strong ordering *per partition* is cheap (single writer per convo — same trick as Kafka partitions and [[Workflow 4 - CQRS and Event Sourcing Service#Step 2 — Build the event store on Postgres|W4's per-aggregate seq]]); global ordering would cost consensus at 700k msg/s.

## Step 5 — Group messages: small-N fan-out

**Do:** design group delivery (≤ 500 members). Compare against [[Workflow 6 - Design a News Feed (Twitter-style)#Step 3 — Fan-out on write vs fan-out on read|the feed's fan-out problem]] — why does chat sit on the opposite side of the trade-off? What breaks at 100k-member channels?

> [!success]- Solution
> Sender → message service persists once per group log → fan-out worker: for each of ≤500 members, registry lookup → push to online sessions, inbox-mark for offline. 500 × registry-lookups per message is fine at chat's write rate.
> Opposite of the feed because **N is bounded and small** (500 vs 50M) and *delivery* (not rendering) is the contract — push-on-write is unambiguously right; there's no celebrity problem by construction (the product capped group size — an *architectural decision expressed as a product rule*, the cheapest kind).
> At 100k-member channels (Telegram-style), you're back in feed territory: stop per-member delivery; members *pull* the channel log on open + lazy push to recently-active members — the hybrid returns. The lesson: **fan-out strategy is a function of N**, not of "chat vs feed."

## Step 6 — Presence, receipts, and typing indicators

**Do:** design presence (online/last-seen) for 500M users. Why must it be lossy/approximate, and what's the anti-pattern to avoid?

> [!success]- Solution
> Heartbeats over the existing websocket (every ~30s) → presence service updates `user: last_seen, status` with TTL; missed 2 heartbeats → offline. Subscriptions: when A opens chat with B, A subscribes to B's presence (pub/sub topic per user, only for *open* conversations — never "all contacts always": 500M × contacts = a firehose nobody displays).
> Presence must be **approximate by design**: it changes at heartbeat frequency × user count (~16M updates/s system-wide) and has zero durability value → in-memory only, lossy pub/sub, no persistence, no ordering. **Anti-pattern:** treating presence with message-grade guarantees (persisted, acked, ordered) — it 10x's system load for data whose value expires in 30 seconds. Grade guarantees per data class ([[02 - Quality Attributes Drills|characteristics]] apply *within* one system, not just across systems). Receipts (delivered/read) ride the normal message path as tiny system-messages — they need the durability presence doesn't.

## Step 7 — Deploys, failures, and the scorecard

**Do:** how do you deploy new chat-server code with 1M connections per server? Then: registry loses a shard; a region's servers die. Write the scorecard.

> [!success]- Solution
> **Deploys:** connection draining — stop accepting, push "reconnect" hints, let clients migrate over ~10 min, then recycle (clients auto-reconnect through the LB to new servers and sync-since-seq — Step 3's pull-as-truth makes deploys safe). Never big-bang restart: 100M simultaneous reconnects = self-inflicted DDoS (thundering herd; stagger with jittered backoff — [[04 - Distributed Styles Drills#Problem 2 Fallacy autopsy|fallacy #1 respects no one]]).
> **Registry shard loss:** routing fails → messages fall back to inbox+push path (degraded latency, zero loss); registry rebuilds from reconnect heartbeats in minutes — because it's a *cache of truth held by clients*, not truth itself.
> **Scorecard:** paid — stateful fleet ops, session registry as tier-0 dependency, per-class delivery semantics to reason about, drain-style deploys. Bought — instant delivery at 700k msg/s, exactly-once appearance on flaky mobile networks, offline correctness for free (inbox), presence at negligible cost. Chat's grand theme: **push is an optimization; pull-from-durable-log is the guarantee.**

---

## Extensions

1. Add end-to-end encryption (Signal-style): which server-side features break (search, server-side media preview, abuse scanning) — and what does that teach about where capability must live?
2. Multi-device (phone + laptop): messages must sync to all of a user's devices — extend the inbox model (per-device cursors on one log).
3. Design message search: client-side index vs server-side — trade-off table under E2E constraints.
4. Estimate storage: 700k msg/s × 1KB × 1 year retention. What's the archival tiering plan (hot/warm/cold)?
