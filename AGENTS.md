# Project rules: Traffic Cop (API gateway)

Follow these rules strictly.

## DO NOT WRITE OR EDIT (I write these by hand)
- Rate-limiting algorithm (token bucket / sliding window counter), both the plain JS and Redis versions
- Lua scripts for atomic counters
- Fail-open / fail-closed logic and the per-route failure policy
- Core request flow: intercept -> check -> forward or 429
- Per-second aggregation logic that feeds the dashboard
- Redis key design (key structure, TTLs)
- README sections on design decisions, tradeoffs, and results

## YOU WRITE THESE
- Docker Compose, GitHub Actions, project scaffolding, config files
- React + Vite dashboard and Socket.io wiring
- Demo API
- autocannon/k6 scripts
- MongoDB/MySQL schema and the blocked-request logging layer
- Redis client/connection setup (not the logic that uses it)
- Boilerplate, linting, formatting, .env handling
- Test harness and scaffolding

## BOUNDARY RULE
- If a task touches both lists, do only your part and stop at the boundary.
- Do not touch files or functions in the "do not write" list, even to fix small bugs or make things compile. Tell me what needs changing and where.
- Do not make design decisions on my parts.

## TODO MARKERS FOR MY PARTS
- When your code reaches a point that depends on something in the "do not write" list, leave a comment in this format and stop:
  // TODO(MAYANK): <what this needs to do, in one line>
- Describe the requirement only (inputs, expected output, which file it belongs in). Do not include the algorithm, pseudocode, or hints on how to implement it.
- Add a placeholder so the project still runs, such as an empty function or a stub that throws "not implemented". Do not fill it with a working implementation.
- Example:
  // TODO(MAYANK): check the rate limit for this ip + route; return { allowed, remaining }
  const result = await checkRateLimit(ip, route); // stub, throws "not implemented"
- At the end of each task, list every TODO(MAYANK) you added with file and line.

## PROJECT STRUCTURE (final)

```
traffic-cop/
├── AGENTS.md
├── README.md
├── .env.example
├── docker-compose.yml                 # [agent] mysql, demo-api, gateway-1, dashboard
├── docker-compose.multi.yml           # [agent] +gateway-2 + nginx
├── .github/workflows/ci.yml           # [agent]
├── docs/
│   ├── 00-main-design.md              # [you] write BEFORE scaffolding
│   ├── 01-rate-limiting.md            # [you] sliding-window + keys + Lua + TTLs
│   ├── 02-failure-behavior.md         # [you] fail-open/closed + async logging policy
│   ├── 03-benchmark-plan.md           # [you]
│   ├── 04-dashboard-data-flow.md      # [you] per-second agg + multi-gateway merge
│   └── decisions.md                   # [you]
├── nginx/nginx.conf                   # [agent] upstream + X-Forwarded-For
├── gateway/
│   ├── package.json / Dockerfile      # [agent]
│   ├── src/
│   │   ├── server.js                  # [agent] trust proxy + express + listen
│   │   ├── proxy.js                   # [agent]
│   │   ├── config.js                  # [agent]
│   │   ├── socketServer.js            # [agent] wiring; merge strategy from doc 04
│   │   ├── redis/client.js            # [agent] connection only
│   │   ├── middleware/gatewayHandler.js   # [you]
│   │   ├── rateLimit/                 # [you]
│   │   │   ├── slidingWindow.js       # [you] only this one stubbed
│   │   │   ├── lua/                   # [you]
│   │   │   ├── keys.js                # [you] key design + TTLs + client-IP resolution
│   │   │   └── policy.js              # [you] per-route limits + fail-open/closed
│   │   ├── aggregation/aggregator.js  # [you]
│   │   └── logging/
│   │       ├── schema.js              # [agent] MySQL table + indexes
│   │       ├── logger.js              # [agent] async, non-blocking; batch/sample per doc 02
│   │       └── cleanup.js             # [agent] periodic delete of old events (MySQL has no TTL)
│   └── tests/
│       ├── helpers/                   # [agent] harness utilities
│       └── (test files are yours)
├── demo-api/
│   ├── package.json / Dockerfile      # [agent]
│   └── src/
│       ├── server.js                  # [agent] /login (strict), /videos (looser)
│       ├── counter.js                 # [agent]
│       └── routes/
│           ├── login.js
│           ├── videos.js
│           └── stats.js               # [agent] GET /__stats, POST /__reset
├── dashboard/                         # React + Vite (agent)
├── loadtest/
│   ├── login.js / videos.js           # [agent]
│   └── results/                      # [you]
└── tests/
    ├── helpers/                       # [agent]
    └── integration/
        └── (test files are yours)
```

### Ownership notes for the structure
- MySQL is the blocked-event store (user chose MySQL over Mongo); no TTL index, so `logging/cleanup.js` handles expiry.
- Only `slidingWindow.js` is stubbed (token bucket is a stretch goal).
- `gatewayHandler.js`, `rateLimit/*`, `aggregation/*` are stubs with `// TODO(MAYANK)` plus throwing placeholders.
- `server.js` handles `trust proxy` for nginx's `X-Forwarded-For`; client-IP resolution logic lives in `keys.js`.
- `socketServer.js` wiring is agent-owned; the multi-gateway merge strategy is defined in `docs/04-dashboard-data-flow.md` (user).
- Test files are user-owned; harness/helpers are agent-owned. Agent only builds what the test files import.
- `00-main-design.md` must be written before scaffolding code.

## DECISIONS STILL PENDING (user-owned docs)
- Algorithm: sliding-window counter (decided).
- Blocked-event store: MySQL (decided).
- Multi-gateway aggregate merge strategy: define in docs/04-dashboard-data-flow.md.
- Async logging policy (batch size, sample rate, retention days): define in docs/02-failure-behavior.md.
- First algorithm stub: slidingWindow.js (decided).
