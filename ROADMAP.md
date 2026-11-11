# Traffic Cop Roadmap

Overview for the agent. MAYANK names one milestone at a time. Do only that milestone, then stop.
M1-M6 = Core. M7 = Next. Stretch only if MAYANK says so (see AGENTS.md).

At the end of every milestone, the agent must: (a) list what was built, (b) list every TODO(MAYANK) added with file and line, (c) stop.

## M0: Design docs
- AGENT: drafts docs/00-main-design.md and docs/decisions.md only, from AGENTS.md and ROADMAP.md. Mark both PROPOSED. Propose concrete numbers for the success criteria (limit, attack rate, tolerance, latency).
- MAYANK: reviews, changes any numbers, approves in chat.
- Done when: MAYANK says "docs approved". Do not scaffold before that.

## M1: Scaffold and plumbing
- AGENT: repo structure per AGENTS.md, .gitignore, .env.example, docker-compose.yml (redis, mysql, demo-api, gateway, dashboard), gateway skeleton (server.js, proxy.js, config.js, redis/client.js), demo-api (login/videos routes, counter, /__stats, /__reset), CI skeleton, and all MAYANK-owned files as stubs with TODO(MAYANK) and throwing placeholders.
- MAYANK: reviews the scaffold, runs it.
- Done when: `docker compose up` starts every service, the demo API answers directly, and every stub is listed.

## M2: Limiter in plain JS
- MAYANK: slidingWindow.js (in-memory version) and the unit test cases.
- AGENT: tests/helpers only (harness utilities).
- Done when: unit tests pass for window boundary and repeated requests.

## M3: Redis + Lua + request flow
- MAYANK: Lua script, keys.js, policy.js (route limits), gatewayHandler.js (intercept -> check -> forward or 429).
- AGENT: nothing beyond the existing Redis client, unless a connection/config fix is requested.
- AGENT (after MAYANK finishes): writes docs/01-rate-limiting.md describing what MAYANK's code actually does, marked PROPOSED until approved.
- Done when: over-limit requests get 429 and /__stats on the demo API shows blocked requests never arrived.

## M4: Failure behavior and logging
- MAYANK: fail-closed on /login in policy.js.
- AGENT: logging/schema.js, logger.js (async, non-blocking, batch/sample), cleanup.js; writes docs/02-failure-behavior.md from the implemented behavior, plus proposed async logging policy details (batch size, sample rate, retention days), marked PROPOSED until approved.
- Done when: with Redis stopped, /login is rejected, and blocked events land in MySQL without slowing requests.

## M5: Load test and measured claim
- AGENT: loadtest/login.js (autocannon or k6 script).
- MAYANK: runs it, interprets the numbers, commits raw output to loadtest/results/.
- Done when: results meet the success criteria in 00-main-design.md.

## M6: Dashboard
- MAYANK: aggregation/aggregator.js (per-second aggregates).
- AGENT: socketServer.js (single gateway), dashboard TrafficChart + BlockedEvents.
- Done when: the dashboard shows allowed vs blocked live during a load test.

## M7: Multi-instance (Next)
- AGENT: nginx/nginx.conf (forward X-Forwarded-For), docker-compose.multi.yml, trust proxy in server.js, loadtest/videos.js.
- MAYANK: fail-open on /videos, client-IP resolution in keys.js, integration test cases proving the limit holds across 2 gateways.
- Done when: the integration test passes against 2 gateways sharing one Redis.

## Stretch
Second algorithm + comparison table, third instance, dashboard merging across instances, CI test for Redis down.
