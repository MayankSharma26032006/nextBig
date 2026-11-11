# Decisions

> **Status: PROPOSED.** Drafted by AGENT for M0. Record of settled and pending decisions.
> Each entry is Context / Decision / Consequence. This file does not contain key
> design, TTL values, Lua, or algorithm internals — those live in MAYANK's code and
> docs 01/02.

## D1 — Implement rate limiting from scratch
- **Context:** the goal is to demonstrate understanding, and AGENTS.md bans rate-limiting libraries.
- **Decision:** no `express-rate-limit`, `rate-limiter-flexible`, or Redis rate-limit helper packages. We implement the limiter ourselves.
- **Consequence:** more code and tests, but full control and explainability.

## D2 — Sliding-window counter
- **Context:** need a counter algorithm that smooths boundary bursts better than a fixed window.
- **Decision:** sliding-window counter; token bucket is a Stretch goal.
- **Consequence:** slightly more state per key; acceptable for the demo.

## D3 — Redis + Lua for atomic counters
- **Context:** concurrent requests, and later multiple gateways, must not race.
- **Decision:** counters live in Redis and are updated atomically via a Lua script.
- **Consequence:** Redis becomes a hard dependency (see failure behavior); one round-trip per check.

## D4 — MySQL for blocked events
- **Context:** need a durable log of blocked requests.
- **Decision:** MySQL with a periodic cleanup job, since MySQL has no TTL index.
- **Consequence:** schema + indexes + a cleanup routine; slightly more ops than a TTL store.

## D5 — Async, non-blocking logging
- **Context:** logging must not slow the request path.
- **Decision:** batched and sampled async writes; the request path never awaits the database.
- **Consequence:** a small chance of losing recent events on crash; acceptable for a demo.

## D6 — Fail-closed on `/login`, fail-open on `/videos`
- **Context:** a sensitive route should not be left open when the limiter is down.
- **Decision:** fail-closed on `/login`; fail-open on `/videos` (M7).
- **Consequence:** a Redis outage rejects logins by design — must be documented as intentional.

## D7 — Dashboard consumes per-second aggregates
- **Context:** pushing every request event over the socket would flood it.
- **Decision:** the gateway aggregates per second; the dashboard receives aggregates only.
- **Consequence:** less granularity, far less traffic.

## D8 — Multi-instance proof via integration test
- **Context:** a dashboard that merges instances is polish, not proof.
- **Decision:** prove shared limits with an integration test (exhaust on gateway A, expect 429 on gateway B); the dashboard stays single-gateway in Core.
- **Consequence:** no aggregate-merge UI; a stronger and simpler proof.

## D9 — Scope tiers
- **Context:** keep the project explainable.
- **Decision:** Core (M1–M6), Next (M7), Stretch. The two-minute, no-AI test decides the tier.
- **Consequence:** several attractive features are deferred.

## D10 — Doc ownership
- **Context:** the blank-page problem versus authorship.
- **Decision:** AGENT drafts `00` and this file; MAYANK approves the numbers. `01` and `02` are written after the code exists and marked PROPOSED until approved.
- **Consequence:** docs track reality, and approval stays with MAYANK.

## Pending / TBD
- Exact limits and window length — MAYANK to approve.
- Async logging batch size / sample rate / retention days — proposed at M4.
- Multi-gateway aggregate merge — only if the dashboard ever merges; otherwise skip.
