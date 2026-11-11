# Traffic Cop — Main Design

> **Status: PROPOSED.** Drafted by AGENT for M0. Awaiting MAYANK approval in chat.
> All numbers below are **targets (hypotheses)**, not measurements. They will be
> replaced at M5 by values taken from `loadtest/results/`. This respects the rule:
> never invent benchmark numbers.

## 1. What this is

A small API gateway that sits in front of a demo API and enforces per-route,
per-client rate limits using shared state in Redis. It is written from scratch
(no rate-limiting libraries) to show the request flow, atomic counter design,
failure behavior, async logging, and load-tested results.

## 2. Goals

- Enforce per-route, per-client limits at the gateway before the request reaches the demo API.
- Share counters across gateway instances via Redis so limits hold horizontally.
- Choose failure behavior explicitly per route (fail-closed vs fail-open).
- Log blocked requests asynchronously without slowing the request path.
- Show live per-second allowed/blocked aggregates on a dashboard.
- Prove the claim with a committed load test.

## 3. Non-goals (Core)

- No token bucket or algorithm comparison (Stretch).
- No multi-instance + nginx until M7.
- No dashboard merging across instances.
- No auth, TLS termination, or production hardening.
- Not a general-purpose proxy; routes are fixed.

## 4. Architecture

Components:

- **Demo API** — self-counts requests; exposes `GET /__stats` and `POST /__reset`.
- **Gateway** — intercepts, checks the limit, forwards or returns 429.
- **Redis** — atomic shared counters.
- **MySQL** — blocked-event log (async writes, periodic cleanup).
- **Dashboard** — React + Vite + Socket.io, per-second aggregates.
- **Load test** — autocannon/k6 scripts plus committed raw results.

Request path (description only, no implementation):

1. Client calls the gateway.
2. Gateway resolves client identity and route.
3. Gateway checks the limit against Redis.
4. Allowed: forward to the demo API. Denied: return 429.
5. Blocked events are queued for async logging; per-second counters update the dashboard.

The rate check, key structure, Lua, and failure policy are MAYANK-owned and live
in `docs/01-rate-limiting.md` and `docs/02-failure-behavior.md`.

## 5. Failure behavior (summary)

- `/login` (sensitive): **fail-closed** — if Redis is unavailable, reject.
- `/videos` (non-sensitive): **fail-open** — if Redis is unavailable, allow. (M7)

Rationale and the async logging policy live in `docs/02-failure-behavior.md`,
written after M4 from the implemented behavior.

## 6. Success criteria (TARGETS — revise at M5)

| Criterion | Target |
|---|---|
| Per-client limit, `/login` | e.g. 5 requests / 60 s |
| Per-client limit, `/videos` | e.g. 60 requests / 60 s |
| Sustained attack rate | e.g. 200 req/s for 30 s |
| Over-limit leakage | ≤ 1% of over-limit requests allowed |
| Added gateway latency | p99 overhead < 5 ms at 500 req/s |
| Blocked requests reaching demo API | 0 |
| Dashboard freshness | ~1 update/second |
| Multi-instance (M7) | same limit holds across 2 gateways on one Redis |

These are placeholders for MAYANK to approve or change. Measured values come
from `loadtest/results/`.

## 7. Scope tiers

Core (M1–M6) → Next (M7) → Stretch. See `ROADMAP.md` for the milestone list.

## 8. Decisions

See `docs/decisions.md`.

## 9. Open questions

- Final limit values and window length — MAYANK to approve.
- Async logging batch size / sample rate / retention days — proposed at M4 in `docs/02-failure-behavior.md`.
