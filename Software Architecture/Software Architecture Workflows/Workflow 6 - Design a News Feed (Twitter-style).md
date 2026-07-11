---
title: "Workflow 6 - Design a News Feed (Twitter-style)"
date: 2026-07-11
tags:
  - software-architecture
  - system-design
  - news-feed
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 6 — Famous Case: News Feed (Twitter/X-style)

> [!abstract] Project
> The most-asked system design interview question. Design a social feed: users post, follow others, and see a timeline of posts from people they follow. The whole problem is one giant read-vs-write trade-off — this case *is* [[Software Architecture Study#9.3 CQRS — Command Query Responsibility Segregation|CQRS]] at planetary scale. Work each step on paper before opening solutions.

**Architecture (target):**

```
  client ──► CDN/edge ──► API Gateway ──► Feed API (stateless)
                                            │
              ┌─────────────────────────────┼──────────────────────┐
              ▼                             ▼                      ▼
        Post service                  Fanout service          Timeline service
        (write path)                  (async workers)         (read path)
              │                             │                      │
        Posts DB (sharded            reads follower           Feed cache: Redis
        by user_id) + outbox ──►Q──► graph, pushes            per-user list of
              │                      post IDs into            post IDs (~800)
        Social graph DB              follower feeds ──────────►
        (follows, sharded)
        Celebrity path: NOT fanned out — merged at read time
```

---

## Step 1 — Requirements & the numbers that shape everything

**Do:** state functional scope (post, follow, timeline; skip DMs/ads) and estimate: 200M DAU, avg user follows 300, checks feed 5×/day, posts 0.3×/day. Compute read/write QPS and the ratio. What characteristic dominates?

> [!success]- Solution
> Writes: 200M × 0.3 / 86,400s ≈ **700 posts/s** (peak ×3 ≈ 2k/s). Reads: 200M × 5 / 86,400 ≈ **11.6k feed-loads/s** (peak ~35k/s) — and each feed-load needs posts from ~300 followees. **Read:write ≈ 16:1 on requests, but ~5000:1 on row-touches** if reads compute naively.
> Dominant characteristics: read **performance** (feed p99 < 200ms — it's the product), **availability** (feed down = product down; posting can degrade), **scalability**. Consistency is explicitly sacrificed: a post appearing 10s late is invisible; nobody audits feed order. This ranking — decided *before* designing — is what makes the rest of the design defensible ([[02 - Quality Attributes Drills#Problem 5 Pick the top 3|drill]]).

## Step 2 — Naive design and why it dies

**Do:** design the obvious version: one SQL DB, `SELECT * FROM posts WHERE author_id IN (SELECT followee FROM follows WHERE follower=?) ORDER BY created_at LIMIT 50`. Compute why it dies at scale, and identify the *two distinct* problems.

> [!success]- Solution
> At 35k feed-loads/s, each query joins the follow list (300 rows) against a posts table of billions of rows, sorted — even perfectly indexed, that's ~10M row-touches/s against one DB. It dies twice: (1) **compute-on-read** — the feed is recalculated from scratch on every load though it barely changed; (2) **single-node ceiling** — no single DB holds this write+read volume. Fixes must therefore be (1) precompute (move work to write time) and (2) partition (shard). Naive-design-first is a *method*: it surfaces which walls you'll hit, so every complexity you add answers a named wall — not fashion ([[Software Architecture Study#12. Solution-Architect Trade-off Analysis|trade-off discipline]]).

## Step 3 — Fan-out on write vs fan-out on read

**Do:** define both strategies, build the trade-off table, then handle the case that breaks each: a celebrity with 50M followers, and a bot following 100k accounts.

> [!success]- Solution
> **Fan-out on write (push):** when a user posts, workers insert the post ID into every follower's precomputed feed list. Reads = one cache lookup — O(1), fast, cheap. Writes = O(followers).
> **Fan-out on read (pull):** store nothing per-follower; at read time, fetch recent posts of all 300 followees and merge. Writes O(1); reads O(followees), slow.
> | | Push | Pull |
> |---|---|---|
> | Read latency | ✅ ~10ms | ❌ 100s of ms |
> | Write cost | O(followers) — celebrity = 50M inserts | ✅ O(1) |
> | Storage | duplicated IDs everywhere | ✅ minimal |
> | Inactive users | ❌ wasted fan-out to dead accounts | ✅ pay only on read |
>
> **The celebrity breaks push** (50M inserts per tweet, minutes of lag, storms on retweet-cascades). **The bot breaks pull** (merging 100k timelines). **Real answer: hybrid** — push for normal users, *don't* fan out celebrities; at read time, merge the user's precomputed feed with a pulled query of the few celebrities they follow (`is_celebrity` flag at ~1M followers threshold). This hybrid is the single most famous trade-off resolution in system design — know it cold.

## Step 4 — Storage & sharding decisions

**Do:** choose storage for: posts, the social graph, the feed cache. For each: technology, shard key, and the query it must make fast. Name the shard-key trap in the posts store.

> [!success]- Solution
> - **Posts:** wide-column/KV (Cassandra-style) or sharded SQL, shard by **author_id** (the pull path asks "recent posts BY user X" — locality per author), post IDs time-sortable (Snowflake IDs encode timestamp → no ORDER BY needed). **Trap:** shard-by-post_id spreads one author's posts across all shards, turning the celebrity pull into a scatter-gather; shard-by-author creates hot shards for celebrities — accept + mitigate with caching, because the celebrity *read* path is cache-friendly (same data for 50M people — perfect cache economics).
> - **Social graph:** sharded KV, shard by follower_id with the *forward* list (who X follows — the read path's question) plus a reverse-index table by followee (who follows X — the fan-out path's question). Two copies, two questions — denormalization is the price of scale.
> - **Feed cache:** Redis, key `feed:{user_id}`, list/zset of ~800 post IDs (cap it — nobody scrolls to 1997), TTL for inactive users, rebuildable from the pull path on miss (cache = disposable projection, [[Workflow 4 - CQRS and Event Sourcing Service#Step 4 — Projections (the CQRS read side)|same principle as W4 projections]]).

## Step 5 — The write path end-to-end

**Do:** trace "user posts" through the system with async boundaries marked. Where does the outbox go? What consistency does the poster see vs their followers?

> [!success]- Solution
> ```
> POST /tweet ──► Post svc: tx{ insert post + outbox } ──► 200 OK (~50ms)
>                    └──relay──► queue ──► Fanout workers (pool):
>                         read follower list (chunked 10k/batch)
>                         LPUSH feed:{follower} for each  (+ trim to 800)
>                         celebrity? skip fanout, done
> ```
> Poster sees their post instantly (read-your-own-writes: their own posts merge into their feed from the posts store directly). Followers see it in seconds→minutes (fan-out lag) — eventual consistency as UX, and here it's *free*: no follower knows when the post "should" have appeared. Compare how much harder this was in [[Workflow 3 - Microservices and Event-Driven on Cloud#Step 5 — The checkout saga across services|W3's checkout]], where the user is *waiting* — same pattern, opposite UX pressure. Idempotent workers + outbox: duplicate fan-out just re-inserts the same post ID (naturally idempotent — set semantics).

## Step 6 — Ranking, ads, and the second read model

**Do:** product wants algorithmic ranking (engagement-based), not chronological. Where does ranking live, and why does this push the design further toward CQRS?

> [!success]- Solution
> Ranking = a *scoring function over candidates*. The feed cache becomes the **candidate pool** (recent posts from followed accounts); at read time a ranker service scores candidates (features: engagement counts, recency, affinity) and orders them. Heavy features are precomputed by stream jobs (Kafka → Flink-style) into a feature store; the read path just does lookup+multiply.
> This is now three read models from one write stream: chronological feed, ranked feed, notification triggers — the event stream (posts, likes, follows) is the source of truth and every product surface is a projection. That's the deep lesson of this case: at scale, **the write model and read models diverge so far that they're different systems** — CQRS isn't a pattern you add, it's what remains when read-optimizing forces you to precompute.

## Step 7 — Failure modes & the scorecard

**Do:** what happens when (a) the fan-out queue backs up 10 minutes, (b) Redis loses a shard, (c) the posts DB has a hot celebrity shard during a world event? Then write the trade-off scorecard.

> [!success]- Solution
> - (a) Feeds go stale — users see old posts, product degrades *gracefully* (nobody 500s). Alert on queue age; autoscale workers. This tolerance is the payoff of choosing availability over consistency in Step 1.
> - (b) Cache misses → rebuild from pull path → thundering-herd risk → per-key rebuild locks + request coalescing; Redis replicas to shrink the blast radius.
> - (c) Hot shard: celebrity posts are identical for everyone → CDN/app-layer cache absorbs reads; write side is fine (one row). World-event *posting* spikes (everyone tweets) → the async write path absorbs it by design (queue depth grows, nothing falls over).
> **Scorecard:** paid — massive storage duplication, seconds-to-minutes delivery lag, operational complexity (workers, streams, caches), eventual consistency everywhere. Bought — O(1) reads at 35k/s, graceful degradation, independent scaling of every stage. The trade was correct *because Step 1 ranked read performance above consistency* — a banking product making the same trade would be malpractice ([[Workflow 4 - CQRS and Event Sourcing Service|W4]] is the counter-case).

---

## Extensions

1. Add "likes count" on every post in the feed — why is this counter surprisingly hard at 35k reads/s, and what's the standard answer? (Sharded counters + periodic aggregation + approximate display.)
2. Design the notification path ("X liked your post") reusing the same event stream — which topology, broker or mediator? ([[04 - Distributed Styles Drills#Problem 6 Broker vs mediator EDA|drill]])
3. A regulator demands deletion propagate within 24h — trace a delete through every copy (posts store, 50M feed caches, CDN, ranker features) and design the tombstone mechanism.
4. Estimate the Redis fleet size for 200M users × 800 post IDs (8 bytes each) + overhead. Is the feed cache affordable? (Show the math.)
