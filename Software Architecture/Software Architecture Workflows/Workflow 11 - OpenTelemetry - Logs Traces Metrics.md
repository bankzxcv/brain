---
title: "Workflow 11 - OpenTelemetry - Logs Traces Metrics"
date: 2026-07-11
tags:
  - software-architecture
  - observability
  - opentelemetry
  - logging
  - tracing
  - metrics
  - workflow
status: in-progress
parent: "[[Software Architecture Study]]"
---

# Workflow 11 — Observability with OpenTelemetry: Logs, Traces, Metrics

> [!abstract] Project
> Instrument the Paidee booking platform from [[Workflow 10 - Saga Patterns Deep Dive|Workflow 10]] (API + orchestrator + 4 services + Kafka) with the **three signals** — logs, traces, metrics — via OpenTelemetry, then use them to solve two production mysteries. Observability appeared as one step in [[Workflow 3 - Microservices and Event-Driven on Cloud#Step 6 — Observability or death|W3]]; here it's the whole project, because in distributed systems **observability is architecture**, not tooling garnish.

**Architecture (target):**

```
 ┌── each service ─────────────────────────┐
 │  app code + OTel SDK                    │
 │   ├─ logs    (structured, trace-linked) │
 │   ├─ traces  (spans, context propagated)│──OTLP──► ┌────────────────┐
 │   └─ metrics (RED + business)           │          │ OTel COLLECTOR │
 └─────────────────────────────────────────┘          │ (receive →     │
        context propagation:                          │  process:batch,│
        HTTP: traceparent header                      │  sample, scrub │
        Kafka: trace headers on messages              │  → export)     │
                                                      └──┬────┬────┬──┘
                                                  traces │    │    │ logs
                                                 (Tempo/ │    │    │ (Loki/
                                                  Jaeger)│    │    │  ES)
                                                     metrics (Prometheus/Mimir)
                                                          │
                                              Grafana: one UI, three signals,
                                              CROSS-LINKED by trace_id/exemplars
```

---

## Step 1 — The three signals, clarified once and for all

**Do:** define each signal by the *question it answers*, its cost profile, and what it's useless for. Most teams get this table wrong and log everything — write it from scratch.

> [!success]- Solution
> | | **Metrics** | **Traces** | **Logs** |
> |---|---|---|---|
> | Question | "**Is** something wrong? How much/many/fast?" | "**Where** in the request path is it wrong?" | "**Why** is it wrong, exactly, for this case?" |
> | Shape | numbers, pre-aggregated (counters, gauges, histograms) | tree of timed **spans**, one tree per request | timestamped structured records (arbitrary detail) |
> | Cost | ~fixed & tiny (aggregation discards detail *at the source*) | grows with traffic → **sampled** | grows with traffic × verbosity → the expensive one |
> | Cardinality | must be LOW (labels multiply series) | high is fine (each trace unique) | anything |
> | Useless for | per-request detail ("which order failed?") | aggregate trends, alerting thresholds | seeing latency structure across services |
> | Typical store | Prometheus/Mimir | Tempo/Jaeger | Loki/Elasticsearch |
>
> The debugging flow is always the same direction: **metric alerts you (is) → trace localizes it (where) → logs explain it (why)**. A team with only logs greps blindly; only metrics knows it's sick but not where; only traces sees shapes without causes. The three are one system — which is OTel's whole thesis: one SDK, one context, one pipeline, correlated signals.

## Step 2 — Structured logs done right

**Do:** define Paidee's log standard: format, mandatory fields, levels policy, and the two classic logging sins to ban. Show one good log line from the payment service.

> [!success]- Solution
> JSON to stdout (the platform ships it — apps never manage log files/rotation):
> ```json
> {"ts":"2026-07-11T09:14:03.221Z","level":"error","service":"payment",
>  "msg":"charge declined","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736",
>  "span_id":"00f067aa0ba902b7","saga_id":"saga-887","trip_id":"trip-1204",
>  "psp":"omise","decline_code":"insufficient_funds","attempt":2,"duration_ms":840}
> ```
> **Mandatory fields:** `ts, level, service, msg` + **`trace_id`/`span_id`** (the bridge to traces — without it, correlation is grep-and-pray) + business correlators (`saga_id`, `trip_id` — what support searches by).
> **Levels policy:** `error` = someone may need to act; `warn` = degraded-but-handled; `info` = state transitions worth an audit line (saga steps!); `debug` = off in prod, toggleable per-service. Alert on error *rates*, never on individual lines.
> **Banned sins:** (1) unstructured prose (`"Something went wrong for user " + id`) — unqueryable; (2) **PII/secrets in logs** (card numbers, tokens, addresses) — a compliance breach with 10-year retention ([[Workflow 5 - Solution Architecture Case Study with Trade-offs|W5's]] auditors will find it); scrub at the Collector as defense-in-depth (Step 5). Bonus sin: logging inside hot loops — that's a metric wearing a log costume, at 1000× the price.

## Step 3 — Distributed tracing & context propagation

**Do:** define spans for one booking request end-to-end (API → orchestrator → flight → hotel → payment, with Kafka hops). Explain trace/span IDs, the `traceparent` header, and the part everyone forgets — propagation through the message broker and the outbox.

> [!success]- Solution
> A **trace** = one request's tree; a **span** = one timed operation with attributes (`span: POST /bookings`, child `span: saga.step.reserve_flight`, child `span: kafka.publish`, …). The **trace_id** (128-bit) rides in the W3C **`traceparent`** HTTP header: `00-{trace_id}-{parent_span_id}-{flags}` — each service continues the same trace and adds its spans. OTel SDKs auto-instrument HTTP/gRPC clients+servers, DB calls, and Kafka producers/consumers; you add manual spans for business steps (`saga.step.*`).
> **The forgotten parts:** (1) **Kafka** — inject `traceparent` into *message headers* at produce, extract at consume, else every async hop starts an orphan trace and your saga is invisible ([[Workflow 3 - Microservices and Event-Driven on Cloud#Step 6 — Observability or death|W3's lesson]]); (2) **the outbox** ([[Workflow 10 - Saga Patterns Deep Dive#Step 5 — Idempotency, duplicates, and the outbox (the plumbing that makes it true)|W10]]) — store the trace context *in the outbox row* so the relay resumes the original trace, not its own; (3) **long async gaps** — a trace spanning a 15-min hold-expiry isn't one request anymore: use **span links** (new trace, linked to the origin) rather than one grotesque 15-minute trace.

## Step 4 — Metrics: RED, USE, and business signals

**Do:** define Paidee's metric set: the RED trio per service, USE for infrastructure, saga/business metrics, and histogram buckets for latency. Then explain the cardinality trap with a concrete bad example.

> [!success]- Solution
> **RED per service** (request-driven systems): `Rate` — requests/s by endpoint+code; `Errors` — error rate; `Duration` — latency **histogram** (buckets tuned to SLO: 50/100/250/500/1000/2500ms — p99 computed from these, and **exemplars** attach sample trace_ids to buckets: click the slow bucket → jump to an actual slow trace).
> **USE for infra** (resource-driven): Utilization / Saturation / Errors per node, DB pool, **Kafka consumer lag** (the async system's heartbeat), queue depth, DLQ size.
> **Business/saga:** `sagas_started/completed/compensated/stuck` (counters), `saga_duration_seconds` (histogram), `bookings_by_status`, `payment_declines_by_code` — the metrics product managers and [[Workflow 10 - Saga Patterns Deep Dive#Step 6 — The ugly cases humans, deadlines, and half-failures|the STUCK alert]] hang off.
> **Cardinality trap:** labels multiply time series. `http_requests_total{endpoint, code}` = ~50 series. Add `user_id` → 2M users × endpoints × codes = **millions of series, Prometheus dies**. Rule: labels are for *bounded* dimensions (endpoint, code, saga_step); unbounded identifiers (user, trip, saga instance) belong in **traces and logs** — that's the signal-choice table from Step 1 enforcing itself economically.

## Step 5 — The Collector: the pipeline is architecture too

**Do:** why put an OTel Collector between apps and backends instead of exporting directly? Design its pipeline (receivers → processors → exporters) including the sampling decision — head vs tail — for a system doing 10k traces/s.

> [!success]- Solution
> Direct export couples every app to backend choice, credentials, retry logic, and cost decisions. The **Collector** decouples: apps speak OTLP to localhost/sidecar; the Collector handles the rest — swap Jaeger→Tempo, add a vendor, change sampling: zero app redeploys ([[Software Architecture Study#7.3 Hexagonal Architecture (Ports & Adapters)|adapter thinking]] applied to telemetry).
> Pipeline: **receivers** (OTLP) → **processors**: batch (efficiency), memory-limiter (protect itself), attribute-scrubber (strip PII — Step 2's defense-in-depth), resource-tagger (cluster/region) → **exporters** (Tempo, Prometheus-remote-write, Loki).
> **Sampling at 10k traces/s** (storing all ≈ pointless money): *Head sampling* (decide at trace start, e.g., 1%) — cheap, but discards 99% of the *interesting* traces since errors are rare. *Tail sampling* (Collector buffers whole traces, then keeps: all errors + all slower-than-SLO + 1% baseline) — exactly the right traces, at the cost of a stateful Collector tier (all spans of a trace must hit the same instance → load-balance by trace_id). Verdict: tail-sample — the whole point of tracing is the anomalies. Errors-and-slow-always, baseline-1% is the industry default.

## Step 6 — Production mystery #1: the latency spike

**Do:** run the workflow: the `booking_duration_seconds` p99 alert fires (4s, SLO 1.5s). Walk the metric→trace→log path to root cause, given: the histogram exemplars point to traces where the `reserve_hotel` span is 3.2s, and hotel-service logs for those trace_ids show `"pool wait", pool_in_use: 50/50`.

> [!success]- Solution
> 1. **Metric (is):** p99 breached; RED per service shows hotel-svc Duration up, Rate flat, Errors ~0 — a *slowness*, not a failure, localized to one service already.
> 2. **Trace (where):** exemplar trace shows the tree: orchestrator span 3.9s → `reserve_hotel` child 3.2s → inside it, `db.query` span only 40ms but starts 3.1s after the span begins → the *gap* before the query is the smoking gun (waiting, not working).
> 3. **Log (why):** trace-linked logs: `"pool wait", pool_in_use: 50/50` — connection-pool exhaustion. Cross-check USE: DB CPU low (it's not the DB), pool saturation 100%.
> Root cause: pool sized 50 while a deploy doubled replicas' traffic share; queries were never slow — requests queued for connections. Fix: raise pool + add `db_pool_wait_seconds` metric + alert on pool saturation ≥ 80% — **converting today's mystery into tomorrow's threshold**, which is how observability compounds. Note what made it 10 minutes instead of a war room: exemplars (metric→trace bridge) and trace_id-in-logs (trace→log bridge). The bridges *are* the system.

## Step 7 — Production mystery #2: the silent saga + the scorecard

**Do:** support reports trips stuck in `PENDING` for hours; no alerts fired. Diagnose using the async-system signals (Step 4), explain why request-oriented monitoring missed it, then write the observability scorecard and cost note.

> [!success]- Solution
> RED was green — every HTTP request succeeded fast. The failure was **between** requests: `sagas_stuck` was never incremented because the orchestrator's relay died silently → check **Kafka consumer lag**: orchestrator's reply-queue lag growing for 6h. No consumption = no progress = no errors. Trace confirms: booking traces end at `kafka.publish`, orphaned — no consumer spans follow.
> **Lesson:** async systems fail by *silence*, not by errors. The mandatory async signals: consumer lag (per group), DLQ depth, outbox backlog age, **saga age** (`now - started_at` histogram; alert p99 > 10 min), and heartbeat/watchdog metrics for every background loop ([[Workflow 10 - Saga Patterns Deep Dive#Step 4 — Build it as orchestration|W10's recovery scan]] should itself emit `last_run_ts` — alert when stale). Every queue in [[Workflow 6 - Design a News Feed (Twitter-style)|W6]]–[[Workflow 9 - Design Ride-Hailing (Uber-style)|W9]] needs the same treatment.
> **Scorecard — paid:** SDK+Collector infra to run; ~5–15% overhead budget (batching + sampling keeps it low); telemetry storage bill (often 10–30% of infra — sample/retain deliberately: traces 7d, metrics 13mo downsampled, logs 30d hot + archive); cardinality discipline forever. **Bought:** minutes-not-days incident resolution across 6 services; SLOs you can actually measure; capacity planning from real histograms; and the audit trail regulators and [[Workflow 5 - Solution Architecture Case Study with Trade-offs|W5-style]] compliance reviews demand. In a monolith, observability is a nice-to-have; past your first network hop it's the difference between engineering and archaeology.

---

## Extensions

1. Define SLOs for Paidee (booking success rate 99.5%, p99 < 1.5s) and implement **error-budget burn-rate alerts** (fast-burn 2% in 1h page; slow-burn 5% in 24h ticket) — why burn-rate beats threshold alerts.
2. Add continuous profiling (Pyroscope/Parca) as the emerging "fourth signal" — which mystery type do the three signals still leave unsolved? (CPU *inside* one span.)
3. Instrument [[Workflow 4 - CQRS and Event Sourcing Service|W4's]] projector: which signal catches projection lag, and what's the exemplar path from a stale-read complaint to the root cause?
4. Cost drill: 10k RPS × 5 services × ~8 spans/request at ~500 bytes/span — compute daily trace volume raw vs tail-sampled (errors+slow+1%), and price both against a $2k/mo observability budget.
